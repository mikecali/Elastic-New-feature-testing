# 🧪 Elastic New Feature Testing

> Hands-on runbooks, configurations, and guides for testing new and beta Elastic Stack features in real environments.

![Elastic Stack](https://img.shields.io/badge/Elastic%20Stack-9.x-005571?style=flat-square&logo=elastic)
![Elastic Cloud](https://img.shields.io/badge/Elastic-Cloud-00BFB3?style=flat-square&logo=elastic)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

---

## 👋 About This Repository

This repo is a personal collection of feature tests, integration guides, and runbooks built while exploring new and beta capabilities of the **Elastic Stack**. Each entry covers a real implementation — tested on live infrastructure, documented with actual commands, real output, and lessons learned along the way.

The goal is simple: go beyond the official docs and share what it actually takes to get things working end to end.

**Who is this for?**
- Elastic practitioners exploring new features before GA
- SREs and platform engineers evaluating Elastic integrations
- Anyone who wants a real-world reference, not just a "hello world"

---

## 📂 Repository Structure

```
Elastic-New-feature-testing/
├── README.md                          ← You are here
│
├── apache-otel/                       ← Apache HTTP Server + OpenTelemetry
│   ├── README.md                      ← Full integration guide
│   └── config/
│       ├── apache-status.conf         ← mod_status config
│       └── otelcol-apache.yaml        ← Standalone collector config (alternative)
│
└── (more features coming...)
```

---

## 🗂 Feature Index

| # | Feature | Area | Stack Version | Status | Guide |
|---|---|---|---|---|---|
| 01 | [Apache HTTP Server — OpenTelemetry Integration](#01-apache-http-server--opentelemetry) | Observability | 9.4.x | ✅ Complete | [→ apache-otel/](./apache-otel/) |

---

## 01 Apache HTTP Server — OpenTelemetry

**What was tested:** Monitoring Apache HTTP Server using the native EDOT (Elastic Distribution of OpenTelemetry) Collector bundled inside Elastic Agent, with the new **Apache HTTP Server OpenTelemetry Input Package** (technical preview), shipping metrics to Elasticsearch and visualizing them in the `[Apache OTel] Overview` Kibana dashboard.

**Why it matters:** Elastic is shifting its infrastructure monitoring story from Metricbeat (ECS format) to native OpenTelemetry. The Apache OTel Input Package is one of the first integrations to deliver this through Fleet — fully managed, no extra software, SLA-covered.

### What's covered

- Enabling `mod_status` and the `/server-status` endpoint
- Installing the Apache OTel Input Package via the Fleet UI (beta)
- Verifying the EDOT Collector pipeline inside Elastic Agent
- Viewing live metrics in the `[Apache OTel] Overview` dashboard
- Known limitations (prefork MPM, `apache.current_connections`)
- Alternative standalone `otelcol-contrib` path with full config

### Key findings

| Finding | Detail |
|---|---|
| **Recommended path** | Apache OTel Input Package via Fleet — no extra processes, Elastic SLA-covered |
| **Alternative path** | Standalone `otelcol-contrib` works but is community-supported only |
| **ES endpoint gotcha** | Must use `.es.` subdomain, not `.kb.` or `.fleet.` |
| **prefork MPM limitation** | `apache.current_connections` not available when using `mod_php` |
| **Dashboard** | `[Apache OTel] Overview` requires `data_stream.dataset: apachereceiver.otel` |
| **Kibana min version** | 9.2.0 required for the Apache OTel Input Package |

### Environment

| Component | Version |
|---|---|
| OS | Debian 12 (Bookworm) |
| Apache | 2.4.67 with mpm_prefork + mod_php 8.2 |
| Elastic Agent | 9.4.2 (Fleet-managed, Elastic Cloud) |
| Elasticsearch | 9.4.1 (Elastic Cloud, us-east-2) |
| Apache OTel Input Package | v0.1.0 (technical preview) |
| Apache OTel Assets Package | v0.5.0 |

📖 **[Full guide →](./apache-otel/README.md)**

---

## 🌐 Test Environment

All features in this repo are tested on **Elastic Cloud (us-east-2)** against real workloads on a Debian 12 server (`mcalizo-o11y-demo`).

```
┌─────────────────────────────┐
│  mcalizo-o11y-demo          │
│  Debian 12 / GCP us-east-2  │
│                             │
│  ├─ Apache 2.4.67           │
│  ├─ Elastic Agent 9.4.2     │
│  └─ (test workloads)        │
└────────────┬────────────────┘
             │ Fleet-managed
             ▼
┌─────────────────────────────┐
│  Elastic Cloud              │
│  us-east-2                  │
│                             │
│  ├─ Elasticsearch 9.4.1     │
│  ├─ Kibana 9.4.x            │
│  └─ Fleet                   │
└─────────────────────────────┘
```

---

## 📋 How Each Guide Is Structured

Every feature guide in this repo follows the same structure so you can jump straight to what you need:

1. **Overview** — what the feature is and why it matters
2. **Prerequisites** — what you need before starting
3. **Step-by-step setup** — exact commands with expected output
4. **Kibana steps** — UI navigation where applicable
5. **Verification** — how to confirm it's actually working
6. **Metrics / output reference** — what data you get
7. **Known limitations** — honest assessment of gaps
8. **Troubleshooting** — real errors encountered and how they were fixed

---

## 🔖 Conventions Used

| Convention | Meaning |
|---|---|
| `<YOUR_ES_ENDPOINT>` | Replace with your Elasticsearch URL |
| `<YOUR_API_KEY>` | Replace with your API key |
| ✅ | Confirmed working in tested environment |
| ⚠️ | Works with caveats or limitations |
| ❌ | Not working / not available in this configuration |

---

## 🗺 Roadmap

Features planned for future testing:

- [ ] System metrics via EDOT OTel (hostmetrics receiver)
- [ ] Nginx OpenTelemetry integration
- [ ] Kubernetes OTel monitoring with EDOT Collector
- [ ] ES|QL for infrastructure observability queries
- [ ] SLO templates with Apache OTel assets
- [ ] Elastic Agent in OTel-only mode (no Metricbeat)

Have a feature you'd like to see tested? Open an issue!

---

## 🤝 Contributing

Found an error, have a better approach, or want to add a new feature test?

1. Fork the repo
2. Create a branch: `git checkout -b feat/your-feature-name`
3. Add your guide following the structure above
4. Update the Feature Index table in this README
5. Open a pull request

---

## ⚠️ Disclaimer

This repository documents personal testing and exploration of Elastic Stack features. It is:

- **Not** officially supported by Elastic
- **Not** a substitute for official Elastic documentation
- Tested against specific versions — results may differ on other versions

Always refer to the [official Elastic documentation](https://www.elastic.co/docs) for production guidance.

---

## 🔗 Useful Links

- [Elastic Documentation](https://www.elastic.co/docs)
- [Elastic Integrations Reference](https://www.elastic.co/docs/reference/integrations)
- [EDOT Collector Docs](https://www.elastic.co/docs/reference/edot-collector)
- [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)
- [Elastic Community Forums](https://discuss.elastic.co)
- [Elastic on GitHub](https://github.com/elastic)

---

## 👤 Author

**Michael Calizo** · [@mikecali](https://github.com/mikecali)

---

*Last updated: June 2026 · Elastic Stack 9.4.x*
