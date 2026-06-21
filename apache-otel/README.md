# 🔭 Apache HTTP Server — OpenTelemetry Integration with Elastic Stack

> Monitor Apache HTTP Server using OpenTelemetry and visualize metrics in Kibana's **[Apache OTel] Overview** dashboard.

![Elastic Stack](https://img.shields.io/badge/Elastic%20Stack-9.4.x-005571?style=flat-square&logo=elastic)
![Apache](https://img.shields.io/badge/Apache-2.4-D22128?style=flat-square&logo=apache)
![OTel](https://img.shields.io/badge/OTel-Native%20EDOT-425CC7?style=flat-square&logo=opentelemetry)
![OS](https://img.shields.io/badge/Debian-12%20Bookworm-A81D33?style=flat-square&logo=debian)
![Support](https://img.shields.io/badge/Elastic%20Support-SLA%20Covered-00BFB3?style=flat-square)

---

## 📋 Table of Contents

- [Recommended Approach](#-recommended-approach)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Step 1 — Enable mod\_status](#step-1--enable-mod_status)
- [Step 2 — Configure the Status Endpoint](#step-2--configure-the-status-endpoint)
- [Step 3 — Install Apache Integration in Fleet (Recommended)](#step-3--install-apache-integration-in-fleet-recommended)
- [Step 4 — Enable the Apache OTel Input Package](#step-4--enable-the-apache-otel-input-package)
- [Step 5 — Verify in Kibana](#step-5--verify-in-kibana)
- [Verify It's Working](#-verify-its-working)
- [Metrics Reference](#-metrics-reference)
- [Known Limitations](#-known-limitations)
- [Troubleshooting](#-troubleshooting)
- [Alternative: Standalone otelcol-contrib](#-alternative-standalone-otelcol-contrib)

---

## 🏆 Recommended Approach

There are **two ways** to ship Apache OTel metrics to Elastic. This README documents **both**, but recommends the native Fleet-managed path:

| | Native EDOT (Recommended) | Standalone otelcol-contrib |
|---|---|---|
| **How** | Apache OTel Input Package via Fleet UI | Separate `otelcol-contrib` process |
| **Managed by** | Elastic Agent / Fleet | systemd service |
| **Elastic Support** | ✅ Full SLA coverage | ⚠️ Community support only |
| **Extra processes** | None — built into Elastic Agent | Requires separate install |
| **Config management** | Fleet UI (no files to edit) | Manual YAML on server |
| **Status** | Beta (Kibana 9.2.0+) | GA |
| **Recommended for** | Production | Testing / multi-backend |

> **TL;DR:** If you already have Elastic Agent enrolled in Fleet, use the **Apache OTel Input Package**. It uses the EDOT Collector bundled inside Elastic Agent — no additional software needed.

---

## 🏗 Architecture

### Native EDOT Path (Recommended)

```
┌─────────────────────┐     ┌──────────────────────────────────────────┐     ┌──────────────────┐
│   Apache 2.4        │     │  Elastic Agent (Fleet-managed)           │     │  Elastic Cloud   │
│                     │     │                                          │     │                  │
│  mod_status         │────▶│  EDOT Collector (bundled)                │────▶│  Elasticsearch   │────▶  Kibana
│  /server-status     │HTTP │  └─ apache receiver (OTel Input Pkg)     │OTLP │  apachereceiver  │      [Apache OTel]
│  ?auto              │     │  └─ resourcedetection/system processor   │     │  .otel dataset   │      Overview
└─────────────────────┘     └──────────────────────────────────────────┘     └──────────────────┘
```

### Standalone otelcol-contrib Path (Alternative)

```
┌─────────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│   Apache 2.4        │     │  otelcol-contrib      │     │  Elastic Cloud   │
│  /server-status     │────▶│  apache receiver      │────▶│  Elasticsearch   │────▶  Kibana
└─────────────────────┘HTTP └──────────────────────┘HTTPS└──────────────────┘
```

---

## ✅ Prerequisites

- Apache 2.4 running on Debian/Ubuntu
- `sudo` / root access on the Apache server
- Elastic Agent enrolled in Fleet (Elastic Cloud) — for the recommended path
- Kibana 9.2.0+ with Stack Management permissions
- Network access from server to Elastic Cloud

---

## Step 1 — Enable mod_status

Check if `mod_status` is already loaded:

```bash
apache2ctl -M | grep status
# Expected: status_module (shared)
```

If not enabled:

```bash
sudo a2enmod status
```

> `mod_status` is enabled by default in most Apache installations. If the command returns `status_module (shared)`, skip to Step 2.

---

## Step 2 — Configure the Status Endpoint

Create the status config file:

```bash
sudo nano /etc/apache2/conf-available/status.conf
```

Paste the following:

```apache
ExtendedStatus On

<Location /server-status>
    SetHandler server-status
    Require local
</Location>
```

Enable and reload:

```bash
sudo a2enconf status
sudo systemctl reload apache2
```

Verify the endpoint is working:

```bash
curl http://localhost/server-status?auto
```

Expected output:

```
ServerVersion: Apache/2.4.67 (Debian)
ServerMPM: prefork
Total Accesses: 250
Total kBytes: 625
BusyWorkers: 2
IdleWorkers: 4
```

> ✅ The presence of `BusyWorkers` and `IdleWorkers` confirms `ExtendedStatus` is active.

---

## Step 3 — Install Apache Integration in Fleet (Recommended)

This step installs the pre-built dashboards, alert rules, and the classic log/metrics collection for Apache — all managed by Fleet.

1. **Kibana → Management → Integrations** → search `Apache HTTP Server`
2. Click **Add Apache HTTP Server**
3. Select your existing **Agent Policy**
4. Set log paths (defaults usually work):
   - Access log: `/var/log/apache2/access.log*`
   - Error log: `/var/log/apache2/error.log*`
5. Set the metrics endpoint: `http://localhost/server-status?auto`
6. Click **Save and deploy changes**

> This deploys the classic Metricbeat-format Apache integration. The OTel-native collection is configured separately in Step 4.

---

## Step 4 — Enable the Apache OTel Input Package

This is the key step that enables **native OTel metric collection** through the EDOT Collector bundled inside your Elastic Agent — no extra software needed.

### Enable Beta Integrations

The Apache OTel Input Package is currently in **technical preview**. To find it:

1. **Kibana → Management → Integrations**
2. Scroll to the bottom and toggle on **"Display beta integrations"**

### Add the Integration

1. Search for **`Apache HTTP Server OpenTelemetry Input Package`**
2. Click **Add Apache HTTP Server OpenTelemetry Input Package**
3. Configure:

| Field | Value |
|---|---|
| **Endpoint** | `http://localhost/server-status?auto` |
| **Enable TLS** | Off (for local HTTP endpoint) |
| **Dataset name** | Leave as default |

4. Under **"Where to add this integration?"** → select **Agent policy 1** (or your policy)
5. Click **Save and deploy changes**

### Verify the Agent Picked It Up

Within ~30 seconds, the agent should show the new Apache OTel pipeline:

```bash
sudo elastic-agent status
```

Look for this in the output — it confirms the native EDOT Apache receiver is running:

```
└─ pipeline:metrics/otelcol-apachereceiver-...
   ├─ status: StatusOK
   ├─ processor:resourcedetection/system/...   StatusOK
   ├─ processor:transform/...-routing          StatusOK
   └─ receiver:apache/...                      StatusOK
```

> ✅ All components showing `StatusOK` means data is flowing to Elasticsearch.

---

## Step 5 — Verify in Kibana

### Check Data in Discover

1. **Kibana → Analytics → Discover**
2. Search: `data_stream.dataset: apachereceiver.otel`
3. Set time range to **Last 15 minutes**
4. Expand a document and confirm fields: `apache.workers.busy`, `apache.requests.total`, `apache.uptime`

### Open the Dashboard

1. **Kibana → Analytics → Dashboards**
2. Search: **`[Apache OTel] Overview`**
3. Use the **Host** dropdown to select your server hostname
4. Set time range to **Last 15 minutes** or **Last 1 hour**

You should see all panels populated:

```
Uptime        Request Rate    Traffic Rate    CPU Load
16 minutes    0.1 req/s       102.4B/s        0.01%

Busy Workers  Idle Workers    Server Load(1m)
1             9               0.02
```

> ⚠️ The **Current Connections** panel will show an error — this is expected with `mpm_prefork`. See [Known Limitations](#-known-limitations).

---

## ✅ Verify It's Working

```bash
# 1. Apache status endpoint responding
curl http://localhost/server-status?auto | head -10

# 2. Elastic Agent healthy and Apache OTel pipeline active
sudo elastic-agent status | grep -A5 "apachereceiver"

# 3. Agent logs — confirm no errors from apache receiver
sudo grep -i apache /var/log/elastic-agent/elastic-agent-$(date +%Y%m%d)*.ndjson | \
  grep -i "error\|fail" | tail -10

# 4. Data count in Elasticsearch (replace placeholders)
curl -s \
  -H "Authorization: ApiKey <YOUR_API_KEY>" \
  "https://<ES_ENDPOINT>/metrics-*/_count?q=data_stream.dataset:apachereceiver.otel"
```

---

## 📊 Metrics Reference

Metrics collected by the Apache OTel receiver via the EDOT Collector:

| Metric | OTel Field | Description | Availability |
|---|---|---|---|
| Uptime | `apache.uptime` | Seconds since last Apache restart | All MPMs |
| Request Rate | `apache.requests.total` | Total requests (derives req/s) | All MPMs |
| Traffic | `apache.traffic.bytes` | Total bytes transferred | All MPMs |
| CPU Load | `apache.cpu.load` | Apache process CPU utilization | All MPMs |
| Busy Workers | `apache.workers.busy` | Active request-handling threads | All MPMs |
| Idle Workers | `apache.workers.idle` | Available worker threads | All MPMs |
| Server Load 1m | `apache.load.1` | 1-minute system load average | All MPMs |
| Server Load 5m | `apache.load.5` | 5-minute system load average | All MPMs |
| Server Load 15m | `apache.load.15` | 15-minute system load average | All MPMs |
| Current Connections | `apache.current_connections` | Active TCP connections | ⚠️ event/worker MPM only |

---

## ⚠️ Known Limitations

### `apache.current_connections` — prefork MPM

If Apache runs with `mpm_prefork` (required by `mod_php`), the `apache.current_connections` metric is **not reported** by `mod_status`. The `[Apache OTel] Overview` dashboard shows an error for this one panel — all other metrics work normally.

**Check your MPM:**
```bash
apache2ctl -V | grep MPM
# Server MPM: prefork  ← current_connections not available
# Server MPM: event    ← current_connections available
```

**To enable `current_connections`**, migrate from `mod_php` to PHP-FPM first, then:
```bash
# Only run AFTER migrating to PHP-FPM
sudo a2dismod mpm_prefork
sudo a2enmod mpm_event
sudo systemctl restart apache2
```

### Apache OTel Input Package Is in Technical Preview

The Fleet-managed Apache OTel Input Package (`apache_input_otel`) requires:
- Kibana **9.2.0+**
- "Display beta integrations" toggle enabled in the Integrations UI

It is developed and maintained by Elastic and expected to reach GA in a future release.

### Ingest Pipelines Not Supported with OTel-Native Data

When using the EDOT/OTel-native ingestion path, Elasticsearch Ingest Pipelines are not applied to the data. If you rely on custom ingest pipelines for field transformation, use the classic Apache integration instead, or apply transformations at the collector level using OTel processors.

---

## 🔧 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Unknown column [apache.uptime]` | No OTel data indexed yet | Wait 30s after deploying, widen time range to Last 15 min |
| `Unknown column [apache.current_connections]` | prefork MPM limitation | Expected — accept or migrate to PHP-FPM + mpm_event |
| Dashboard shows N/A for all panels | Time range too narrow or wrong host selected | Set to Last 15 min; select host in Host dropdown |
| `403` on `/server-status` | `Require local` not matching client IP | Add `Require ip 127.0.0.1 ::1` to Location block |
| Agent status shows no `apachereceiver` pipeline | Apache OTel Input Package not deployed | Add the integration via Fleet UI (Step 4) |
| Agent status shows `apachereceiver` but no data | Endpoint URL wrong in integration config | Edit integration, verify endpoint is `http://localhost/server-status?auto` |
| `flush failed (404)` (otelcol-contrib path) | Wrong ES endpoint (`.fleet.` or `.kb.` subdomain) | Use `https://<name>.es.<region>.aws.elastic-cloud.com` |

---

## 📁 File Locations

```
# Apache
/etc/apache2/conf-available/status.conf     # Server-status endpoint config
/var/log/apache2/access.log                 # Apache access logs
/var/log/apache2/error.log                  # Apache error logs

# Elastic Agent
/etc/elastic-agent/elastic-agent.yml        # Agent base config (Fleet-managed)
/var/log/elastic-agent/                     # Agent logs
/var/lib/elastic-agent/data/               # Agent data directory (EDOT collector lives here)

# Alternative: Standalone otelcol-contrib
/etc/otelcol-contrib/config.yaml            # OTel Collector pipeline config
/lib/systemd/system/otelcol-contrib.service # OTel Collector systemd unit
```

---

## 🔗 References

- [Apache HTTP Server OTel Input Package — Elastic Docs](https://www.elastic.co/docs/reference/integrations/apache_input_otel)
- [Apache HTTP Server OTel Assets — Elastic Docs](https://www.elastic.co/docs/reference/integrations/apache_otel)
- [EDOT Collector — Elastic Docs](https://www.elastic.co/docs/reference/edot-collector)
- [EDOT vs Upstream OpenTelemetry](https://www.elastic.co/docs/reference/opentelemetry/compatibility/edot-vs-upstream)
- [OTel Apache Receiver — GitHub](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/apachereceiver)
- [Apache mod_status Documentation](https://httpd.apache.org/docs/2.4/mod/mod_status.html)

---

## 🔄 Alternative: Standalone otelcol-contrib

> Use this path if you need community collector flexibility, multi-backend export, or are testing without Elastic Agent.

<details>
<summary>Click to expand standalone otelcol-contrib setup</summary>

### Install

```bash
curl -Lo /tmp/otelcol-contrib.deb \
  https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.146.1/otelcol-contrib_0.146.1_linux_amd64.deb

sudo dpkg -i /tmp/otelcol-contrib.deb
otelcol-contrib --version
```

### Configure

```bash
sudo tee /etc/otelcol-contrib/config.yaml << 'EOF'
receivers:
  apache:
    endpoint: http://localhost/server-status?auto
    collection_interval: 10s

processors:
  resourcedetection/system:
    detectors: ["system"]
    system:
      hostname_sources: ["os"]
      resource_attributes:
        host.name:
          enabled: true
        host.id:
          enabled: false

exporters:
  elasticsearch/otel:
    endpoint: https://<YOUR_ES_ENDPOINT>
    api_key: <YOUR_API_KEY>
    mapping:
      mode: otel

service:
  pipelines:
    metrics:
      receivers: [apache]
      processors: [resourcedetection/system]
      exporters: [elasticsearch/otel]
EOF
```

> ⚠️ Use the `.es.` subdomain for the ES endpoint, **not** `.kb.` or `.fleet.`

### Start

```bash
sudo systemctl enable otelcol-contrib
sudo systemctl start otelcol-contrib
sudo journalctl -u otelcol-contrib -f | grep -i "elastic\|error\|apache"
```

### Get Your API Key

1. **Kibana → Stack Management → Security → API Keys → Create API key**
2. Name: `otel-apache-collector`
3. Privileges:
```json
{
  "otel-apache": {
    "indices": [{
      "names": ["metrics-*", "logs-*"],
      "privileges": ["auto_configure", "create_doc"]
    }]
  }
}
```
4. Copy the **Encoded** (Base64) value

</details>

---

## 🧪 Environment

Tested on:

- **OS:** Debian 12 (Bookworm) — `mcalizo-o11y-demo`
- **Apache:** 2.4.67 with `mpm_prefork` and `mod_php 8.2`
- **Elastic Agent:** 9.4.2 (Fleet-managed, Elastic Cloud)
- **EDOT Collector:** bundled with Elastic Agent 9.4.2
- **Elasticsearch:** 9.4.1 (Elastic Cloud, us-east-2)
- **Apache OTel Input Package:** v0.1.0 (technical preview)
- **Apache OTel Assets Package:** v0.5.0
