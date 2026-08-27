---
title: "Dashboards & Visualisation"
weight: 6
---

# Dashboards & Visualisation

Dashboards transform raw telemetry into actionable insights. A well-designed dashboard answers the question "is everything OK?" in under five seconds and guides investigation when it isn't. This section covers Grafana, dashboard design principles, and the established methodologies for choosing what to visualise.

## Methodologies: What to Measure

### The Four Golden Signals

Defined in the Google SRE book, the four golden signals are the minimum set of metrics for any user-facing service:

| Signal | What It Measures | Example Metric |
|--------|-----------------|----------------|
| **Latency** | Time to serve a request | `http_request_duration_seconds` (histogram) |
| **Traffic** | Demand on the system | `http_requests_total` (counter) |
| **Errors** | Rate of failed requests | `http_requests_total{status=~"5.."}` (counter) |
| **Saturation** | How "full" the service is | CPU %, memory %, queue depth, thread pool usage |

**Rule:** If you can only have one dashboard per service, make it a golden signals dashboard.

### The RED Method

Developed by Tom Wilkie (Grafana Labs), RED is a simplification of the golden signals for request-driven services:

| Metric | Definition | PromQL Example |
|--------|-----------|----------------|
| **R**ate | Requests per second | `sum(rate(http_requests_total[5m]))` |
| **E**rrors | Failed requests per second | `sum(rate(http_requests_total{status=~"5.."}[5m]))` |
| **D**uration | Latency distribution | `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))` |

RED is ideal for microservices. Apply it to every service, and you have a consistent view across your architecture.

### The USE Method

Developed by Brendan Gregg, USE focuses on infrastructure resources:

| Metric | Definition | What to Check |
|--------|-----------|---------------|
| **U**tilisation | % of resource capacity used | CPU %, memory %, disk I/O %, network bandwidth % |
| **S**aturation | Work queued because resource is full | CPU run queue length, swap usage, disk I/O wait |
| **E**rrors | Error events on the resource | ECC memory errors, NIC packet drops, disk errors |

USE is ideal for infrastructure dashboards (nodes, databases, message brokers).

### Choosing the Right Method

| System Type | Method | Focus |
|-------------|--------|-------|
| Microservices, APIs | RED | Request health |
| Infrastructure (CPU, memory, disk) | USE | Resource health |
| Mixed / SRE overview | Golden Signals | Both |

## Grafana

Grafana is the dominant open-source visualisation platform. It supports 100+ data sources (Prometheus, Elasticsearch, CloudWatch, Datadog, Tempo, Loki) and provides a unified dashboarding experience.

### Dashboard JSON Structure

Every Grafana dashboard is defined as JSON. Understanding the structure lets you version-control dashboards and generate them programmatically.

```json
{
  "dashboard": {
    "title": "Order Service — RED",
    "uid": "order-service-red",
    "tags": ["production", "order-service"],
    "timezone": "utc",
    "refresh": "30s",
    "time": { "from": "now-1h", "to": "now" },
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{service=\"order-service\"}[5m]))",
            "legendFormat": "requests/s"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "custom": { "fillOpacity": 10, "lineWidth": 2 }
          }
        }
      },
      {
        "title": "Error Rate",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{service=\"order-service\", status=~\"5..\"}[5m])) / sum(rate(http_requests_total{service=\"order-service\"}[5m])) * 100",
            "legendFormat": "error %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 1 },
                { "color": "red", "value": 5 }
              ]
            }
          }
        }
      },
      {
        "title": "Latency (P50, P95, P99)",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 24, "x": 0, "y": 8 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{service=\"order-service\"}[5m])))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{service=\"order-service\"}[5m])))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service=\"order-service\"}[5m])))",
            "legendFormat": "P99"
          }
        ],
        "fieldConfig": {
          "defaults": { "unit": "s" }
        }
      }
    ]
  }
}
```

### Dashboard as Code with Grafonnet

Grafonnet is a Jsonnet library for generating Grafana dashboards programmatically:

```jsonnet
local grafana = import 'github.com/grafana/grafonnet/gen/grafonnet-latest/main.libsonnet';
local dashboard = grafana.dashboard;
local panel = grafana.panel;
local prometheus = grafana.query.prometheus;

dashboard.new('Order Service — RED')
+ dashboard.withUid('order-service-red')
+ dashboard.withTags(['production', 'order-service'])
+ dashboard.withRefresh('30s')
+ dashboard.withPanels([
  panel.timeSeries.new('Request Rate')
  + panel.timeSeries.queryOptions.withTargets([
    prometheus.new('prometheus',
      'sum(rate(http_requests_total{service="order-service"}[5m]))')
    + prometheus.withLegendFormat('requests/s'),
  ])
  + panel.timeSeries.standardOptions.withUnit('reqps')
  + panel.timeSeries.gridPos.withW(12)
  + panel.timeSeries.gridPos.withH(8),
])
```

### Panel Types

| Panel Type | Use For | Example |
|-----------|---------|---------|
| Time Series | Metrics over time (rate, latency, throughput) | Request rate, error rate |
| Stat | Single current value with thresholds | Current error %, uptime |
| Gauge | Progress toward a target | SLO budget remaining |
| Bar Gauge | Ranked comparison | Top 5 endpoints by latency |
| Table | Detailed tabular data | Active alerts, recent deploys |
| Heatmap | Distribution over time | Latency distribution buckets |
| Logs | Log lines from Loki/Elasticsearch | Correlated error logs |
| Traces | Trace visualisation from Tempo/Jaeger | Selected trace waterfall |

## Dashboard Design Principles

### The 5-Second Rule

A viewer should understand the system's health within 5 seconds of opening the dashboard. This means:

1. **Top row: status summary** — Stat panels with colour-coded thresholds (green/yellow/red)
2. **Middle: time series** — Rate, errors, and latency over time
3. **Bottom: details** — Tables, logs, or breakdowns by endpoint/service

### Layout Pattern

```text
┌──────────────────────────────────────────────────────┐
│  [STAT]        [STAT]        [STAT]        [STAT]    │  ← Status at a glance
│  Requests/s    Error Rate    P99 Latency   CPU %     │
├──────────────────────────┬───────────────────────────┤
│  [TIME SERIES]           │  [TIME SERIES]            │  ← Trends
│  Request Rate            │  Error Rate %             │
├──────────────────────────┴───────────────────────────┤
│  [TIME SERIES — full width]                          │  ← Latency distribution
│  Latency: P50, P95, P99                             │
├──────────────────────────────────────────────────────┤
│  [TABLE]                                             │  ← Detail
│  Top 10 slowest endpoints                            │
└──────────────────────────────────────────────────────┘
```

### Design Rules

| Rule | Why |
|------|-----|
| Use consistent time ranges across all panels | Prevents misleading correlations |
| Set meaningful thresholds (green/yellow/red) | Enables instant status recognition |
| Use units on all axes (ms, %, req/s) | Prevents misinterpretation |
| Limit to 6–8 panels per dashboard | Cognitive overload reduces comprehension |
| Use template variables for service/environment | One dashboard definition serves all services |
| Include annotation overlays for deploys | Correlate behaviour changes with code changes |
| Title every panel clearly | "P99 Latency (ms)" not "Panel 1" |

### Template Variables

Template variables make dashboards reusable across services and environments:

```text
Variable: $service
Query: label_values(http_requests_total, service)
Type: Query

Variable: $environment
Values: production, staging, development
Type: Custom

# Used in panel queries:
sum(rate(http_requests_total{service="$service", environment="$environment"}[5m]))
```

### Annotations

Annotations overlay events (deploys, config changes, incidents) on time series:

```json
{
  "annotations": {
    "list": [
      {
        "name": "Deployments",
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "expr": "changes(deploy_timestamp{service=\"$service\"}[1m]) > 0",
        "tagKeys": "version",
        "iconColor": "blue"
      }
    ]
  }
}
```

When a latency spike aligns with a deploy annotation, root cause is immediately apparent.

## Dashboard Hierarchy

Organise dashboards in layers:

| Level | Dashboard | Audience | Content |
|-------|-----------|----------|---------|
| L0 | Platform Overview | On-call, management | All services health, SLO status |
| L1 | Service Dashboard | Service owner | RED metrics, dependencies, resources |
| L2 | Debug Dashboard | Debugging engineer | Per-endpoint breakdown, logs, traces |
| L3 | Infrastructure | Platform team | Node CPU/memory/disk, Kubernetes pods |

Drill-down links connect the layers: click an unhealthy service in L0 to open its L1 dashboard, click a slow endpoint in L1 to open L2 with pre-filtered logs.

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Dashboard sprawl (hundreds of dashboards) | Nobody knows which to use | Curate a small set of canonical dashboards |
| Averages instead of percentiles | Hides tail latency | Always show P50, P95, P99 |
| No thresholds | Green lines mean nothing without context | Set meaningful thresholds based on SLOs |
| Raw counters on graphs | Meaningless rising lines | Apply `rate()` before visualising counters |
| Missing units | "42" — 42 what? | Set units on every panel axis |
| Too many series per panel | Spaghetti graph, unreadable | Limit to 5–7 series or use `topk()` |

## Key Takeaways

- Use the RED method (Rate, Errors, Duration) for request-driven services and the USE method (Utilisation, Saturation, Errors) for infrastructure — the golden signals combine both
- A well-designed dashboard communicates system health within 5 seconds: stat panels at the top, time series in the middle, detail tables at the bottom
- Version-control dashboards as JSON or generate them with Grafonnet — never rely solely on UI-created dashboards that live only in the Grafana database
- Always visualise percentiles (P50, P95, P99) rather than averages for latency — averages mask tail latency problems that affect real users
- Template variables make one dashboard serve all services and environments — use `label_values()` queries for dynamic population
- Deploy annotations on time series graphs are the single most useful correlation tool — they instantly link behaviour changes to code changes
