# Elasticsearch Index Growth Monitoring

A step-by-step guide for tracking index size over time using Kibana Lens and the Stack Monitoring cluster.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1  Verify Monitoring Data Availability](#step-1--verify-monitoring-data-availability)
- [Step 2  Identify Your Largest Indices](#step-2--identify-your-largest-indices)
- [Step 3  Build the Kibana Lens Dashboard](#step-3--build-the-kibana-lens-dashboard)
- [Step 4  Query Historical Growth](#step-4--query-historical-growth)
- [Understanding Cluster Scope](#understanding-cluster-scope)
- [Troubleshooting](#troubleshooting)
- [Next Steps](#next-steps)

---

## Overview

This guide explains how to monitor the growth of Elasticsearch indices over time using:

- The **Stack Monitoring cluster** (where historical index stats are stored)
- **Kibana Lens** (for time-series visualisation)
- **Dev Tools** (for ad-hoc queries)

> **Important:** `GET /_cat/indices` only shows the *current* state of your indices. It cannot be used to compare sizes over time. Use the monitoring cluster queries in this guide instead.

**Example Dashboard**
<img width="1913" height="493" alt="Screenshot 2026-06-23 at 12 27 59 PM" src="https://github.com/user-attachments/assets/54b8271f-01aa-44e0-ae86-9875083728f4" />

---

## Prerequisites

| Requirement | Details |
|---|---|
| Stack Monitoring enabled | Metricbeat or legacy monitoring shipping ES stats to a dedicated monitoring cluster |
| Kibana access | Kibana pointed at the monitoring cluster |
| Dev Tools access | Ability to run queries against the monitoring cluster |
| Monitoring data retention | At least as long as your comparison window (e.g. 30 days) |

---

## Step 1 — Verify Monitoring Data Availability

Before building your dashboard, confirm how much historical data exists. Run this in **Dev Tools on your monitoring cluster**:

```json
GET .monitoring-es-*/_search
{
  "size": 1,
  "sort": [{ "@timestamp": { "order": "asc" } }],
  "_source": ["@timestamp"]
}
```

The returned `@timestamp` is the oldest data point available. If your retention is shorter than your desired comparison window, extend it before proceeding.

---

## Step 2 — Identify Your Largest Indices

Run this in **Dev Tools on your data cluster** (the cluster holding your business data):

```json
GET /_cat/indices?v&h=index,store.size,pri.store.size,docs.count&s=store.size:desc&bytes=gb&size=20
```

Note the index names or patterns you want to track. Common patterns:

| Pattern | Description |
|---|---|
| `your-data-index-*` | Your business/application data |
| `.ds-metricbeat-*` | Metricbeat data streams |
| `.ds-.monitoring-es-8-mb-*` | ES monitoring data |
| `logs-*`, `traces-*`, `metrics-*` | Fleet/APM data streams |

> **Tip:** Data streams roll over into multiple backing indices (e.g. `.ds-metricbeat-9.3.0-2026.04.20-000003`). The newest backing index will be actively growing; older ones will be flat.

---

## Step 3 — Build the Kibana Lens Dashboard

### 3.1 Open Lens

1. In Kibana, go to **Dashboards → Create dashboard**
2. Click **Add panel → Lens**
3. Set the index pattern to `.monitoring-es-8-*` (or `.monitoring-es-*` for older deployments)

### 3.2 Configure the Visualization

| Setting | Value |
|---|---|
| Chart type | **Line** |
| Horizontal axis | `@timestamp` — Minimum interval: **Day** |
| Vertical axis | `max(index_stats.total.store.size_in_bytes)` |
| Value format | **Bytes (1024)** — auto-displays as KB/MB/GB |
| Breakdown | Top N values of `index_stats.index` |
| Rank by | `max(index_stats.total.store.size_in_bytes)`  Descending |

### 3.3 Apply a KQL Filter

In the **KQL bar at the top of the Lens editor**, enter a filter scoped to the indices you want to track:

```
index_stats.index: (your-index-name OR your-other-index-* OR another-index)
```

> ⚠️ **Put this filter in the KQL bar at the top of the screen  not in the Breakdown "Include values" field.** The breakdown field does not accept KQL syntax and will return no results.

Example to exclude noisy system indices:

```
index_stats.index: * AND NOT index_stats.index: .apm* AND NOT index_stats.index: .kibana*
```

### 3.4 Interpreting the Chart

| Line behaviour | Meaning |
|---|---|
| Flat line | Rolled-over / closed backing index, no longer receiving writes |
| Upward slope | Active write index, currently growing |
| Gap before a certain date | Monitoring data only exists from that point onwards |
| Multiple lines same data stream | Each rollover slice appears as a separate line |

---

## Step 4  Query Historical Growth

To get a programmatic size comparison over a time window, run this aggregation in **Dev Tools on the monitoring cluster**:

```json
GET .monitoring-es-*/_search
{
  "size": 0,
  "query": {
    "range": { "@timestamp": { "gte": "now-30d" } }
  },
  "aggs": {
    "by_index": {
      "terms": { "field": "index_stats.index", "size": 50 },
      "aggs": {
        "size_at_start": {
          "min": { "field": "index_stats.total.store.size_in_bytes" }
        },
        "size_now": {
          "max": { "field": "index_stats.total.store.size_in_bytes" }
        },
        "doc_count_now": {
          "max": { "field": "index_stats.total.docs.count" }
        }
      }
    }
  }
}
```

The difference between `size_now` and `size_at_start` is the **growth delta** for each index over the selected window.

Adjust the `gte` value to change the comparison window:

| Value | Window |
|---|---|
| `now-7d` | Last 7 days |
| `now-30d` | Last 30 days |
| `now-90d` | Last 90 days |
| `2026-01-01` | From a specific date |

---

## Understanding Cluster Scope

A common point of confusion — `_cat/indices` and the Lens dashboard are querying **different clusters**. This is expected and correct.

| Query / Tool | Which Cluster |
|---|---|
| `GET /_cat/indices` | Your **data cluster** (holds your business data) |
| Lens dashboard (`.monitoring-es-*`) | Your **monitoring cluster** (Kibana pointed here) |
| Growth aggregation queries | Your **monitoring cluster** (Dev Tools) |
| `GET /_remote/info` | Either cluster — checks Cross-Cluster Search config |

The monitoring cluster stores historical stats *about* your data cluster, which is what enables growth trending over time.

---

## Troubleshooting

| Symptom | Resolution |
|---|---|
| No results after applying KQL filter | Ensure filter is in the **top KQL bar**, not in Breakdown "Include values" |
| Y-axis shows raw bytes (e.g. `40,000,000,000`) | Set Value format on the vertical axis metric to **Bytes (1024)** |
| All lines are flat | You may be looking at rolled-over indices. The active write index will slope upward |
| Data only appears from a recent date | Monitoring was started recently — data only goes back as far as retention allows |
| Top 10 shows only system/APM indices | Add `NOT index_stats.index: .apm*` to your KQL filter |
| Chart shows stacked bars instead of individual lines | Change chart type from **Bar/Stacked** to **Line** |
| Breakdown ranked alphabetically | Change **Rank by** in Breakdown settings to your metric, Descending |

---

## Next Steps

- **Add a doc count panel** — duplicate your Lens panel and change the metric to `max(index_stats.total.docs.count)` to track document growth alongside size
- **Aggregate data streams** — instead of tracking individual backing indices, use:
  ```json
  GET /_data_stream/<your-data-stream-name>/_stats
  ```
- **Set up a Kibana alert** — trigger a notification when an index exceeds a size threshold
- **Review ILM policies** — use the growth rates you've identified to tune Index Lifecycle Management rollover and delete phases
- **Extend monitoring retention** — if you need longer historical windows, increase the monitoring index retention period in Stack Management

---

## Reference

- [Elasticsearch Stack Monitoring docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/monitor-elasticsearch-cluster.html)
- [Kibana Lens docs](https://www.elastic.co/guide/en/kibana/current/lens.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [Data streams](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html)
