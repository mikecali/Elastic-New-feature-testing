# 🛡️ Defend for Containers (D4C) — Kubernetes Runtime Security with Elastic

> Deploy Elastic's **Defend for Containers** integration on a self-managed Kubernetes cluster (k3s) to get real-time container workload protection using eBPF — detecting process execution, file changes, and container drift.

![Elastic Stack](https://img.shields.io/badge/Elastic%20Stack-9.4.x-005571?style=flat-square&logo=elastic)
![Kubernetes](https://img.shields.io/badge/k3s-v1.35.5-326CE5?style=flat-square&logo=kubernetes)
![eBPF](https://img.shields.io/badge/eBPF-LSM%20%2B%20Tracepoints-orange?style=flat-square)
![OS](https://img.shields.io/badge/Ubuntu-24.04%20%7C%20Kernel%206.8-E95420?style=flat-square&logo=ubuntu)
![Status](https://img.shields.io/badge/Status-Working-brightgreen?style=flat-square)

---

## 📋 Table of Contents

- [What It Does](#-what-it-does)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Step 1 — Verify Kernel Requirements](#step-1--verify-kernel-requirements)
- [Step 2 — Enable eBPF LSM at Boot](#step-2--enable-ebpf-lsm-at-boot)
- [Step 3 — Install k3s](#step-3--install-k3s)
- [Step 4 — Deploy Elastic Agent DaemonSet](#step-4--deploy-elastic-agent-daemonset)
- [Step 5 — Configure Fleet and Add D4C Integration](#step-5--configure-fleet-and-add-d4c-integration)
- [Step 6 — Verify It's Working](#step-6--verify-its-working)
- [Step 7 — Trigger and Observe Detections](#step-7--trigger-and-observe-detections)
- [What You See in Kibana](#-what-you-see-in-kibana)
- [Known Limitations](#-known-limitations)
- [Troubleshooting](#-troubleshooting)
- [Config Reference](#-config-reference)

---

## 🔍 What It Does

Defend for Containers (D4C) uses **eBPF LSM hooks and tracepoints** to monitor container workloads at the kernel level — before system calls complete. It streams telemetry to Elasticsearch and can:

| Capability | How |
|---|---|
| **Process telemetry** | Stream all fork/exec events from containers |
| **Drift detection** | Alert when executables are created or modified inside a running container |
| **Drift prevention** | Block executable changes (policy-configurable) |
| **File monitoring** | Track file create/modify/delete inside containers |
| **Interactive session detection** | Flag when someone `exec`s into a container interactively |
| **SIEM integration** | Feed Elastic Security prebuilt rules for container threat detection |

---

## 🏗 Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Ubuntu 24.04 Host (Kernel 6.8 + eBPF LSM enabled)      │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  k3s (Kubernetes v1.35.5)                       │    │
│  │                                                 │    │
│  │  ┌─────────────────────┐  ┌─────────────────┐  │    │
│  │  │ elastic-agent pod   │  │ nginx-test pod  │  │    │
│  │  │ (DaemonSet)         │  │ (workload)      │  │    │
│  │  │                     │  │                 │  │    │
│  │  │ ┌─────────────────┐ │  │ cp /bin/ls      │  │    │
│  │  │ │ cloud-defend    │◀─────/usr/bin/evil   │  │    │
│  │  │ │ (eBPF probes)   │ │  │ ← DETECTED      │  │    │
│  │  │ └────────┬────────┘ │  └─────────────────┘  │    │
│  │  └──────────┼──────────┘                        │    │
│  └─────────────┼───────────────────────────────────┘    │
│                │ HTTPS                                   │
└────────────────┼────────────────────────────────────────┘
                 ▼
    ┌────────────────────────┐
    │  Elastic Cloud         │
    │  Elasticsearch 9.4.x   │
    │  logs-cloud_defend.*   │
    │                        │
    │  Kibana Security       │
    │  → Kubernetes Dashboard│
    │  → Alerts              │
    └────────────────────────┘
```

---

## ✅ Prerequisites

- Ubuntu 22.04+ or Debian 12 (x86_64)
- Kernel **5.10+** (tested on 6.8.0)
- Elastic Cloud deployment with Fleet enabled
- `sudo` / root access
- Internet access (to pull k3s and Elastic Agent images)

---

## Step 1 — Verify Kernel Requirements

D4C needs eBPF LSM, BTF, and specific capabilities. Check them all:

```bash
# Kernel version (needs 5.10+)
uname -r

# eBPF kernel config — all must be =y
grep -E "CONFIG_BPF|CONFIG_LSM|CONFIG_DEBUG_INFO_BTF" /boot/config-$(uname -r)

# BTF vmlinux file (required for eBPF programs)
ls -la /sys/kernel/btf/vmlinux && echo "BTF available"

# Required capabilities in bounding set
capsh --print | grep -i "sys_resource\|perfmon\|bpf"
```

**Required output:**
```
CONFIG_BPF=y
CONFIG_BPF_LSM=y
CONFIG_DEBUG_INFO_BTF=y
BTF available
```

---

## Step 2 — Enable eBPF LSM at Boot

The kernel compiles in BPF LSM support but doesn't activate it by default. You must add it to the boot parameters:

```bash
# Check current active LSMs
cat /sys/kernel/security/lsm
# Default: lockdown,capability,landlock,yama,apparmor  (no 'bpf')

# Add bpf to the GRUB kernel command line
sudo sed -i \
  's/GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 lsm=lockdown,capability,landlock,yama,apparmor,bpf"/' \
  /etc/default/grub

# Verify the change
grep GRUB_CMDLINE /etc/default/grub

# Apply and reboot
sudo update-grub
sudo reboot
```

After reboot, verify:

```bash
cat /sys/kernel/security/lsm
# Must include: ...,apparmor,bpf
```

> ⚠️ Without `bpf` in the active LSM stack, D4C can log events but **cannot block** them.

---

## Step 3 — Install k3s

D4C requires Kubernetes with containerd as the container runtime. k3s is the fastest way to get a single-node cluster running:

```bash
# Install k3s (single-node Kubernetes)
curl -sfL https://get.k3s.io | sh -

# Set up kubeconfig
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config

# Verify node is Ready
kubectl get nodes
# NAME          STATUS   ROLES           AGE   VERSION
# ubuntu-test   Ready    control-plane   2m    v1.35.5+k3s1

# Verify containerd socket (required by D4C)
ls -la /run/k3s/containerd/containerd.sock
```

> k3s uses containerd by default — this is exactly what D4C requires (containerd or CRI-O).

---

## Step 4 — Deploy Elastic Agent DaemonSet

D4C runs as part of the Elastic Agent deployed as a DaemonSet. The key requirements are:

- **Image:** `elastic-agent-complete` (standard image does not include `cloud-defend`)
- **Capabilities:** `BPF`, `PERFMON`, `SYS_RESOURCE`
- **Env var:** `HOSTFS_PROC_PATH=/hostfs/proc`
- **Volume mounts:** `/boot`, `/sys/fs/bpf`, `/sys/kernel/security` (D4C-specific)

Create the namespace and apply the manifest:

```bash
kubectl create namespace elastic-agent
kubectl apply -f elastic-agent-d4c.yaml
kubectl get pods -n elastic-agent -w
```

The pod should reach `Running` status in ~60-90 seconds (image is ~1GB).

See [elastic-agent-d4c.yaml](#-config-reference) below for the full manifest.

---

## Step 5 — Configure Fleet and Add D4C Integration

### Get enrollment token from Kibana

1. **Kibana → Fleet → Enrollment tokens → Create enrollment token**
2. Name: `d4c-k3s`
3. Select your Agent Policy
4. Copy the token — paste it into `elastic-agent-d4c.yaml` as `FLEET_ENROLLMENT_TOKEN`

### Add Defend for Containers integration

Once the agent enrolls (check **Fleet → Agents** — should show `ubuntu-test` as Healthy):

1. **Fleet → Agent Policies → [your policy] → Add integration**
2. Search **"Defend for Containers (BETA)"**
3. Click **Add Defend for Containers**
4. Leave default policy settings (includes process telemetry + drift detection)
5. **Save and deploy changes**

Fleet pushes the D4C config to the agent within ~30 seconds.

### Verify D4C loaded

```bash
POD=$(kubectl get pods -n elastic-agent -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n elastic-agent $POD -- elastic-agent status 2>/dev/null | grep -i "cloud\|defend"
```

Expected:
```
├─ cloud_defend/control-default
│  └─ status: (HEALTHY)
```

---

## Step 6 — Verify It's Working

### Check index templates are installed

In **Kibana → Dev Tools**:

```
GET /_index_template/logs-cloud_defend*
```

Should return three templates:
- `logs-cloud_defend.file`
- `logs-cloud_defend.process`
- `logs-cloud_defend.alerts`

### Check data in Discover

**Kibana → Analytics → Discover** — set time to **Today** or **Last 2 hours**:

```
data_stream.dataset: cloud_defend.process
```

You should immediately see process events from all running containers including the elastic-agent pod itself.

### Check the Kubernetes Security Dashboard

**Kibana → Security → Dashboards → Kubernetes**

You should see:
- **Clusters:** 1
- **Nodes:** 1
- **Pods:** running count
- **Container images:** list of images
- **Session interactivity:** interactive vs non-interactive sessions

---

## Step 7 — Trigger and Observe Detections

### Deploy a test workload

```bash
kubectl run nginx-test --image=nginx --restart=Never
kubectl wait --for=condition=Ready pod/nginx-test --timeout=60s
```

### Trigger drift detection (copy a binary = executable change)

```bash
kubectl exec -it nginx-test -- bash -c "cp /bin/ls /usr/bin/ls-drift"
kubectl exec -it nginx-test -- bash -c "cp /bin/cat /usr/local/bin/cat-evil"
```

### Trigger interactive session detection

```bash
kubectl exec -it nginx-test -- bash -c "whoami && id"
```

### Check for detections in Kibana

**Security → Dashboards → Kubernetes** — you should see:
- `nginx-test` pod in the container list
- Interactive sessions counted
- Process entries for `cp`, `bash`, etc.

**Discover → `data_stream.dataset: cloud_defend.file`** — drift events from the binary copies.

**Security → Alerts** — prebuilt rules fire for executable creation in containers.

---

## 🖥 What You See in Kibana

### Kubernetes Security Dashboard

```
Clusters    Namespace    Nodes    Pods    Container images
    1           2          1        2           2

Session interactivity          Entry session users
  Interactive:    2              Root:     4
  Non-interactive: 2             Non-root: 0

Container image                          Session count
docker.io/library/nginx                       3
docker.elastic.co/elastic-agent/elastic-...   2
```

### Process Telemetry (cloud_defend.process)

Each event includes:
- `process.executable` — full path of executed binary
- `process.interactive` — whether it was an interactive session
- `container.image.name` — which container
- `orchestrator.resource.name` — pod name
- `event.action` — `fork` or `exec`

### File Events (cloud_defend.file)

Triggered by the drift commands:
- `event.action` — `creation`, `modification`, `deletion`
- `file.path` — `/usr/bin/ls-drift`
- `process.executable` — `/usr/bin/cp` (what created it)

---

## ⚠️ Known Limitations

### Cloud provider detection warning

On self-managed (non-cloud) clusters, D4C logs:
```
could not detect cloud provider — cloud.* fields will be empty
```
This is expected and non-blocking. D4C works fully — the `cloud.*` ECS fields are just empty.

### Cluster shown as "unknown" in dashboard

Since k3s is self-managed, the cluster name shows as `unknown` in the Kubernetes dashboard. This is because cloud metadata (used to determine cluster identity) isn't available. All other data is fully functional.

### Minimum kernel version

D4C requires **kernel 5.10.16+**. The `bpf` LSM hook point for blocking was introduced in Linux 5.8 but the full feature set requires 5.10.16.

---

## 🔧 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `input not supported — correct flavor` | Using standard `elastic-agent` image | Switch to `elastic-agent-complete` image |
| `cloud_defend` not in agent status | D4C integration not added to policy | Fleet → Agent Policies → Add integration → Defend for Containers |
| `elasticsearch index error: index is missing` | Index templates not installed | Re-add D4C integration from Fleet — this triggers template installation |
| `could not detect cloud provider` | Self-managed cluster, no cloud metadata | Expected — non-blocking, `cloud.*` fields will be empty |
| `cloud_defend HEALTHY → STOPPED (exit code 1)` | Missing volume mounts (`/boot`, `/sys/fs/bpf`, `/sys/kernel/security`) | Use the complete DaemonSet manifest below |
| `bpf` not in `/sys/kernel/security/lsm` | eBPF LSM not activated at boot | Add `lsm=...,bpf` to GRUB and reboot |
| Pod `ErrImagePull` | Wrong image path (old: `beats/elastic-agent`) | Use `docker.elastic.co/elastic-agent/elastic-agent-complete:9.4.2` |
| `FindEnrollmentAPIKey: hit count mismatch` | Stale enrollment token | Create a new token in Kibana Fleet → Enrollment tokens |

---

## 📁 Config Reference

### elastic-agent-d4c.yaml

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: elastic-agent
  namespace: elastic-agent
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: elastic-agent
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - namespaces
      - events
      - pods
      - services
      - configmaps
      - serviceaccounts
      - persistentvolumes
      - persistentvolumeclaims
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "deployments", "replicasets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes/stats", "nodes/proxy", "nodes/metrics"]
    verbs: ["get"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterrolebindings", "clusterroles", "rolebindings", "roles"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: elastic-agent
subjects:
  - kind: ServiceAccount
    name: elastic-agent
    namespace: elastic-agent
roleRef:
  kind: ClusterRole
  name: elastic-agent
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: elastic-agent
  namespace: elastic-agent
  labels:
    app: elastic-agent
spec:
  selector:
    matchLabels:
      app: elastic-agent
  template:
    metadata:
      labels:
        app: elastic-agent
    spec:
      serviceAccountName: elastic-agent
      hostNetwork: true
      hostPID: true
      dnsPolicy: ClusterFirstWithHostNet
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - name: elastic-agent
          # IMPORTANT: must use elastic-agent-complete — standard image lacks cloud-defend
          image: docker.elastic.co/elastic-agent/elastic-agent-complete:9.4.2
          env:
            - name: FLEET_ENROLL
              value: "1"
            - name: FLEET_URL
              value: "<YOUR_FLEET_URL>"
            - name: FLEET_ENROLLMENT_TOKEN
              value: "<YOUR_ENROLLMENT_TOKEN>"
            - name: FLEET_INSECURE
              value: "false"
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: ELASTIC_NETINFO
              value: "false"
            # Required for cloud-defend to read host process list
            - name: HOSTFS_PROC_PATH
              value: "/hostfs/proc"
          securityContext:
            runAsUser: 0
            capabilities:
              add:
                - BPF       # Load eBPF programs (Linux 5.8+)
                - PERFMON   # Attach eBPF tracepoints (Linux 5.8+)
                - SYS_RESOURCE  # Modify rlimit_memlock for eBPF maps
                - NET_ADMIN
                - SYS_PTRACE
                - DAC_READ_SEARCH
          resources:
            requests:
              memory: "500Mi"
              cpu: "100m"
            limits:
              memory: "1200Mi"
              cpu: "1000m"
          volumeMounts:
            - name: proc
              mountPath: /hostfs/proc
              readOnly: true
            - name: cgroup
              mountPath: /hostfs/sys/fs/cgroup
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: etc-full
              mountPath: /hostfs/etc
              readOnly: true
            - name: var-lib
              mountPath: /hostfs/var/lib
              readOnly: true
            - name: etc-mid
              mountPath: /etc/machine-id
              readOnly: true
            - name: sys-kernel-debug
              mountPath: /sys/kernel/debug
            # The following three are required specifically for cloud-defend (D4C)
            - name: boot
              mountPath: /boot
              readOnly: true
            - name: sys-fs-bpf
              mountPath: /sys/fs/bpf
            - name: sys-kernel-security
              mountPath: /sys/kernel/security
              readOnly: true
            # k3s containerd socket (instead of /var/run/docker.sock)
            - name: containerd-sock
              mountPath: /run/k3s/containerd/containerd.sock
            - name: elastic-agent-state
              mountPath: /usr/share/elastic-agent/state
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: cgroup
          hostPath:
            path: /sys/fs/cgroup
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: varlog
          hostPath:
            path: /var/log
        - name: etc-full
          hostPath:
            path: /etc
        - name: var-lib
          hostPath:
            path: /var/lib
        - name: etc-mid
          hostPath:
            path: /etc/machine-id
            type: File
        - name: sys-kernel-debug
          hostPath:
            path: /sys/kernel/debug
        # Required for cloud-defend (D4C)
        - name: boot
          hostPath:
            path: /boot
        - name: sys-fs-bpf
          hostPath:
            path: /sys/fs/bpf
        - name: sys-kernel-security
          hostPath:
            path: /sys/kernel/security
        # k3s containerd socket
        - name: containerd-sock
          hostPath:
            path: /run/k3s/containerd/containerd.sock
        - name: elastic-agent-state
          hostPath:
            path: /var/lib/elastic-agent-state
            type: DirectoryOrCreate
```

---

## 🔗 References

- [Defend for Containers — Elastic Docs](https://www.elastic.co/docs/solutions/security/cloud/d4c/d4c-overview)
- [Getting Started with D4C — Elastic Security Labs](https://www.elastic.co/security-labs/getting-started-with-defend-for-containers)
- [cloud_defend Integration Reference](https://www.elastic.co/docs/reference/integrations/cloud_defend)
- [Elastic Agent Managed Kubernetes — Official YAML](https://github.com/elastic/elastic-agent/blob/main/deploy/kubernetes/elastic-agent-managed-kubernetes.yaml)
- [k3s Installation](https://docs.k3s.io/quick-start)

---

## 🧪 Environment

Tested on:

- **OS:** Ubuntu 24.04.3 LTS (`ubuntu-test`)
- **Kernel:** 6.8.0-124-generic (eBPF LSM enabled via GRUB)
- **Kubernetes:** k3s v1.35.5+k3s1 (containerd 2.2.3)
- **Elastic Agent:** 9.4.2 (`elastic-agent-complete` image)
- **Elasticsearch:** 9.4.1 (Elastic Cloud, us-east-2)
- **D4C Integration:** v1.4.0 (BETA)

---

*Last updated: June 2026 · Elastic Stack 9.4.x*
