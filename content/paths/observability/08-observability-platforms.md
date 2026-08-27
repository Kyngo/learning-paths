---
title: "Observability Platforms"
weight: 8
---

# Observability Platforms

Choosing an observability platform is one of the most impactful infrastructure decisions a team makes. The platform determines how you collect, store, query, and alert on telemetry — and it significantly affects your monthly bill. This section compares the major platforms: Datadog, Grafana Cloud, New Relic, and AWS-native tools.

## Platform Architecture Models

There are two fundamental approaches:

### All-in-One SaaS

A single vendor handles metrics, logs, traces, and dashboards. You send telemetry to their endpoints and use their UI.

**Examples:** Datadog, New Relic, Dynatrace, Splunk Observability

| Advantage | Disadvantage |
|-----------|-------------|
| Unified UI — correlate signals without switching tools | Vendor lock-in (proprietary agents, query languages) |
| Managed infrastructure — no TSDB, no Elasticsearch | Expensive at scale (per-host, per-GB pricing) |
| Fast setup — minutes to first dashboard | Limited control over data retention and processing |
| Built-in ML/anomaly detection | Data egress can be complex |

### Composable Open Source (Grafana Stack)

Assemble best-of-breed tools for each signal and unify them in Grafana:

```text
Metrics:  Prometheus / Mimir
Logs:     Loki
Traces:   Tempo
Profiles: Pyroscope
Dashboard: Grafana
Alerting:  Grafana Alerting / Alertmanager
```

| Advantage | Disadvantage |
|-----------|-------------|
| No vendor lock-in — use OpenTelemetry, swap backends | Operational overhead — you manage the infrastructure |
| Cost-effective at scale (open-source, own storage) | Integration between tools requires configuration |
| Full control over data, retention, and processing | Steeper learning curve |
| Grafana Cloud offers a managed option | Feature parity with SaaS varies |

## Datadog

Datadog is the market-leading all-in-one observability platform.

### Capabilities

| Feature | Details |
|---------|---------|
| Metrics | Custom metrics, infrastructure metrics, 800+ integrations |
| Logs | Log Management with pattern clustering, Logging without Limits™ |
| Traces | APM with distributed tracing, flame graphs, service maps |
| Profiling | Continuous Profiler (CPU, memory, lock, I/O) |
| Synthetics | Browser tests, API tests, multistep |
| RUM | Real User Monitoring for web and mobile |
| Security | Cloud Security Posture Management, ASM |
| Dashboards | Drag-and-drop, notebooks, SLO tracking |

### Agent Configuration

The Datadog Agent runs on each host and collects telemetry:

```yaml
# datadog.yaml
api_key: <DATADOG_API_KEY>
site: datadoghq.eu                    # EU datacenter
hostname: order-service-prod-01
tags:
  - env:production
  - service:order-service
  - team:checkout

logs_enabled: true
apm_config:
  enabled: true
  env: production

process_config:
  enabled: true
```

### Datadog with OpenTelemetry

Datadog supports OTLP ingestion — you can send OTel data directly:

```yaml
# otel-collector-config.yaml
exporters:
  datadog:
    api:
      key: ${DD_API_KEY}
      site: datadoghq.eu
    metrics:
      resource_attributes_as_tags: true
    traces:
      span_name_as_resource_name: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog]
```

### Pricing Model

Datadog prices per host, per GB, and per metric:

| Component | Pricing Basis | Notes |
|-----------|--------------|-------|
| Infrastructure | Per host/month | Includes metrics, dashboards, alerts |
| APM (traces) | Per host/month + per span | Ingestion and retention costs |
| Log Management | Per GB ingested + indexed | "Logging without Limits" archives cheaply, indexing is expensive |
| Custom Metrics | Per 100 custom metrics/month | This adds up fast with high cardinality |
| Synthetics | Per test run | API tests cheaper than browser tests |

**Cost trap:** Custom metrics are the most common surprise cost. Each unique metric name + tag combination counts as a custom metric. A single poorly-labelled metric can generate thousands of custom metrics.

## Grafana Cloud

Grafana Labs offers a managed version of the Grafana open-source stack.

### Components

| Component | Underlying Technology | Purpose |
|-----------|----------------------|---------|
| Grafana | Grafana | Dashboards, explore, alerting |
| Mimir | Prometheus-compatible TSDB | Long-term metrics storage |
| Loki | Loki | Log aggregation (label-indexed, not full-text) |
| Tempo | Tempo | Distributed tracing (object storage backend) |
| Pyroscope | Pyroscope | Continuous profiling |
| OnCall | Grafana OnCall | Incident management, escalation |
| k6 | k6 | Load testing and synthetic monitoring |

### Configuration

```yaml
# Prometheus remote_write to Grafana Cloud Mimir
remote_write:
  - url: https://prometheus-prod-01-eu-west-0.grafana.net/api/prom/push
    basic_auth:
      username: <GRAFANA_CLOUD_USER_ID>
      password: <GRAFANA_CLOUD_API_KEY>

# OTel Collector exporting to Grafana Cloud
exporters:
  otlphttp/grafana:
    endpoint: https://otlp-gateway-prod-eu-west-0.grafana.net/otlp
    headers:
      Authorization: "Basic <BASE64_USER:API_KEY>"
```

### Loki Query Language (LogQL)

Loki uses a Prometheus-inspired query language:

```logql
# All error logs from order-service
{service="order-service"} |= "ERROR"

# Parse JSON and filter by field
{service="order-service"} | json | level="ERROR" | duration_ms > 5000

# Error rate from logs
sum(rate({service="order-service"} |= "ERROR" [5m]))

# Top 5 error messages
{service="order-service"} | json | level="ERROR"
| line_format "{{.message}}"
| topk(5, sum by (message) (count_over_time({service="order-service"} | json | level="ERROR" [1h])))
```

### Pricing Model

Grafana Cloud uses usage-based pricing:

| Component | Pricing Basis | Free Tier |
|-----------|--------------|-----------|
| Metrics | Per active series/month | 10,000 series |
| Logs | Per GB ingested | 50 GB/month |
| Traces | Per GB ingested | 50 GB/month |
| Profiles | Per GB ingested | 10 GB/month |
| Alerting | Included | Included |

**Cost advantage:** Loki indexes only labels (not full-text), making log storage 10–100× cheaper than Elasticsearch-based solutions. Tempo uses object storage (S3/GCS), making trace storage extremely cheap.

## New Relic

New Relic positions itself as a full-stack observability platform with a consumption-based pricing model.

### Capabilities

| Feature | Details |
|---------|---------|
| APM | Auto-instrumentation for Java, .NET, Node, Python, Go, Ruby, PHP |
| Infrastructure | Host monitoring, Kubernetes, cloud integrations |
| Logs | Log management with pattern detection |
| Distributed Tracing | Full trace analysis, service maps |
| Browser | Real user monitoring |
| Synthetics | Scripted browser and API tests |
| NRQL | Powerful SQL-like query language |
| Alerts | AI-powered anomaly detection |

### NRQL (New Relic Query Language)

NRQL is SQL-like and very intuitive:

```sql
-- P99 latency by service over the last hour
SELECT percentile(duration, 99) FROM Transaction
WHERE appName = 'order-service'
SINCE 1 hour ago FACET name

-- Error rate
SELECT percentage(count(*), WHERE error IS true) FROM Transaction
WHERE appName = 'order-service'
SINCE 1 hour ago TIMESERIES

-- Deployment impact
SELECT average(duration), percentage(count(*), WHERE error IS true)
FROM Transaction WHERE appName = 'order-service'
SINCE 1 hour ago COMPARE WITH 1 hour ago
```

### Pricing Model

| Component | Pricing Basis | Notes |
|-----------|--------------|-------|
| Data | Per GB ingested/month | All data types: metrics, logs, traces, events |
| Users | Per full-platform user/month | Basic users are free |

**Advantage:** Simple pricing model — one rate per GB regardless of data type. No per-host fees.

**Disadvantage:** Full-platform users are expensive; organisations with many engineers accessing the UI pay significantly more.

## AWS Native Observability

For AWS-centric workloads, native tools integrate deeply with the platform.

### CloudWatch

```text
CloudWatch Metrics → CloudWatch Alarms → SNS → Lambda/PagerDuty
CloudWatch Logs → Logs Insights (query) → Dashboards
CloudWatch Container Insights → ECS/EKS metrics
```

CloudWatch Logs Insights:

```text
fields @timestamp, @message, service, level
| filter level = "ERROR"
| stats count() as errors by service
| sort errors desc
| limit 10
```

### AWS X-Ray

```text
Application (X-Ray SDK) → X-Ray Daemon → X-Ray Service → Console
                                                        → Service Map
                                                        → Trace Analysis
```

### AWS Managed Grafana and Prometheus

AWS offers managed versions of both:

- **Amazon Managed Service for Prometheus (AMP):** Prometheus-compatible TSDB, PromQL, remote_write compatible
- **Amazon Managed Grafana (AMG):** Grafana with SSO integration, data source management

### AWS Pricing

| Service | Pricing Basis | Notes |
|---------|--------------|-------|
| CloudWatch Metrics | Per metric/month + API calls | First 10 free; custom metrics add up |
| CloudWatch Logs | Per GB ingested + stored + queried | Logs Insights charged per GB scanned |
| X-Ray | Per trace recorded + retrieved | Sampling reduces cost |
| AMP | Per metric sample ingested + stored + queried | Similar to Grafana Cloud |
| AMG | Per active editor/month | Viewer access is cheaper |

## Platform Comparison

| Feature | Datadog | Grafana Cloud | New Relic | AWS Native |
|---------|---------|---------------|-----------|------------|
| Setup complexity | Low | Medium | Low | Low (for AWS) |
| Vendor lock-in | High | Low (open standards) | Medium | High (AWS-only) |
| Query language | Proprietary | PromQL, LogQL, TraceQL | NRQL (SQL-like) | CloudWatch Insights |
| OpenTelemetry support | Good (OTLP ingestion) | Excellent (native) | Good (OTLP ingestion) | Partial |
| Trace-to-log correlation | Excellent | Good | Excellent | Limited |
| Custom metrics cost | Expensive | Moderate | Included in per-GB | Moderate |
| Log storage cost | Expensive (indexed) | Low (Loki label-only) | Moderate (per-GB) | Moderate |
| Best for | Teams wanting all-in-one | Open-source-first teams | Simple pricing model | AWS-only workloads |

## Decision Framework

| If you... | Consider |
|-----------|---------|
| Want the fastest setup with least operational overhead | Datadog or New Relic |
| Are cost-sensitive at scale (>100 hosts) | Grafana Cloud or self-hosted Grafana stack |
| Run exclusively on AWS | AWS native tools + Amazon Managed Grafana |
| Want to avoid vendor lock-in | Grafana Cloud with OpenTelemetry |
| Need advanced ML/anomaly detection out of the box | Datadog or New Relic |
| Have a strong platform engineering team | Self-hosted Grafana + Prometheus + Loki + Tempo |

## Key Takeaways

- All-in-one platforms (Datadog, New Relic) offer the fastest time-to-value but create vendor lock-in and can become expensive at scale — composable stacks (Grafana Cloud) offer more control
- Custom metrics are the most common cost surprise on Datadog; log indexing is the second — plan your labelling strategy before instrumenting
- Grafana Cloud's use of Loki (label-indexed logs) and Tempo (object-storage traces) makes it significantly cheaper for log and trace storage than full-text-indexed alternatives
- Always instrument with OpenTelemetry regardless of your backend choice — it decouples your application code from the platform and makes future migration possible
- AWS native tools are cost-effective for AWS-only workloads but lack the cross-cloud and advanced correlation features of dedicated platforms
- Pricing models vary dramatically: Datadog charges per host + per metric + per GB; New Relic charges per GB + per user; Grafana Cloud charges per active series + per GB — model your expected volume before committing
