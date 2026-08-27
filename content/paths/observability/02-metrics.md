---
title: "Metrics"
weight: 2
---

# Metrics

Metrics are numeric measurements collected at regular intervals and aggregated over time. They are the most efficient observability signal — compact to store, fast to query, and ideal for dashboards and alerting. This section covers metric types, the Prometheus data model, PromQL fundamentals, and naming conventions.

## Metric Types

There are four fundamental metric types. Each serves a different purpose.

### Counter

A counter is a cumulative value that only goes up (or resets to zero on restart). Use counters for things you *count*: requests, errors, bytes transferred.

```text
http_requests_total{method="GET", status="200"} 48293
http_requests_total{method="POST", status="500"} 17
```

You never read a counter's raw value directly. Instead, you compute the *rate* — how fast it's increasing:

```promql
# Requests per second over the last 5 minutes
rate(http_requests_total[5m])

# Error rate as a percentage
rate(http_requests_total{status=~"5.."}[5m])
/ rate(http_requests_total[5m]) * 100
```

**Rule:** Never use a counter to track something that can decrease (e.g., queue depth, active connections). That's a gauge.

### Gauge

A gauge is a value that can go up or down. Use gauges for current state: temperature, memory usage, active connections, queue depth.

```text
node_memory_AvailableBytes 8589934592
process_open_fds 42
queue_depth{queue="orders"} 1583
```

Gauges are read directly or aggregated:

```promql
# Current memory usage across all instances
sum(node_memory_AvailableBytes)

# Maximum queue depth in the last hour
max_over_time(queue_depth{queue="orders"}[1h])
```

### Histogram

A histogram samples observations (typically request durations or response sizes) and counts them in configurable buckets. It also tracks the total sum and count.

```text
http_request_duration_seconds_bucket{le="0.01"}  24054
http_request_duration_seconds_bucket{le="0.05"}  33024
http_request_duration_seconds_bucket{le="0.1"}   35102
http_request_duration_seconds_bucket{le="0.5"}   35891
http_request_duration_seconds_bucket{le="1"}     35992
http_request_duration_seconds_bucket{le="+Inf"}  36001
http_request_duration_seconds_sum                 5765.432
http_request_duration_seconds_count               36001
```

Histograms let you compute percentiles server-side:

```promql
# P99 latency over the last 5 minutes
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# P50 (median) latency by endpoint
histogram_quantile(0.50,
  sum by (endpoint, le) (rate(http_request_duration_seconds_bucket[5m]))
)

# Average request duration
rate(http_request_duration_seconds_sum[5m])
/ rate(http_request_duration_seconds_count[5m])
```

**Bucket selection matters.** Bad buckets produce inaccurate percentiles. For HTTP services, these are reasonable defaults:

```text
[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
```

### Summary

A summary is similar to a histogram but computes quantiles client-side (in the application) rather than server-side. It exposes pre-calculated percentiles:

```text
http_request_duration_seconds{quantile="0.5"}  0.042
http_request_duration_seconds{quantile="0.9"}  0.089
http_request_duration_seconds{quantile="0.99"} 0.245
http_request_duration_seconds_sum              5765.432
http_request_duration_seconds_count            36001
```

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Quantile accuracy | Approximate (depends on buckets) | Precise (computed in client) |
| Aggregation | Can aggregate across instances | Cannot aggregate quantiles |
| CPU cost (client) | Low | Higher (streaming quantile) |
| Flexibility | Query any percentile post-hoc | Fixed quantiles at instrumentation |
| Recommendation | **Prefer histograms** | Use only when exact client-side quantiles are required |

**Best practice:** Use histograms in almost all cases. They can be aggregated across instances, and Prometheus native histograms (introduced in v2.40) improve accuracy without bucket explosion.

## The Prometheus Data Model

Prometheus is the de facto standard for metrics collection. Its data model is simple but powerful.

### Time Series

Every time series is uniquely identified by its metric name and a set of key-value labels:

```text
<metric_name>{<label_name>=<label_value>, ...}  <value>  <timestamp>
```

Example:

```text
api_http_requests_total{method="POST", handler="/api/orders", status="201"} 762 1724745600
```

### Label Cardinality

Each unique combination of label values creates a new time series. This is the single most important cost factor in Prometheus:

```text
# 4 methods × 20 endpoints × 5 status codes = 400 time series
api_http_requests_total{method, handler, status}

# Adding user_id as a label: 4 × 20 × 5 × 100,000 users = 40,000,000 series
# DO NOT DO THIS — it will crash Prometheus
```

**Rule:** Never use unbounded values (user IDs, request IDs, email addresses) as metric labels. Use traces or logs for high-cardinality dimensions.

### Metric Naming Conventions

Prometheus naming conventions are widely adopted:

| Rule | Example | Notes |
|------|---------|-------|
| Snake_case, lowercase | `http_request_duration_seconds` | Not camelCase |
| Include unit as suffix | `_seconds`, `_bytes`, `_total` | Always use base units (seconds not milliseconds) |
| Counters end in `_total` | `http_requests_total` | Mandatory for counters |
| Use a descriptive prefix | `myapp_orders_created_total` | Namespace by application or domain |
| Avoid label names in metric name | ❌ `http_get_requests_total` | Use `http_requests_total{method="GET"}` |

Good vs bad naming:

```text
# Good
http_request_duration_seconds        (histogram)
http_requests_total                  (counter)
process_resident_memory_bytes        (gauge)
order_processing_duration_seconds    (histogram)

# Bad
httpRequestDuration                  (camelCase, no unit)
request_count                        (ambiguous, missing _total)
latency_ms                           (use base units: seconds)
http_get_request_count               (method in name, not label)
```

## PromQL Fundamentals

PromQL is Prometheus's query language. It operates on time series data using four value types: instant vectors, range vectors, scalars, and strings.

### Instant Vector Selectors

Return the most recent value for each matching series:

```promql
# All HTTP request counters
http_requests_total

# Filter by label
http_requests_total{status="500"}

# Regex match
http_requests_total{status=~"5.."}

# Negative match
http_requests_total{method!="OPTIONS"}
```

### Range Vector Selectors

Return all values within a time window (required for functions like `rate`):

```promql
# All values from the last 5 minutes
http_requests_total[5m]
```

### Common Functions

```promql
# Rate: per-second increase of a counter over a window
rate(http_requests_total[5m])

# Increase: total increase over a window (rate × window)
increase(http_requests_total[1h])

# Sum: aggregate across dimensions
sum by (method) (rate(http_requests_total[5m]))

# Top 5 endpoints by request rate
topk(5, sum by (handler) (rate(http_requests_total[5m])))

# Percentile from histogram
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Predict value 4 hours from now (linear extrapolation)
predict_linear(node_filesystem_avail_bytes[6h], 4 * 3600)
```

### Practical PromQL Recipes

**Error rate percentage:**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m])) * 100
```

**Apdex score** (satisfied < 0.3s, tolerating < 1.2s):

```promql
(
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
  + sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m]))
) / 2
/ sum(rate(http_request_duration_seconds_count[5m]))
```

**Saturation — CPU usage per instance:**

```promql
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

**Disk will be full in 24 hours:**

```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 24*3600) < 0
```

## Prometheus Architecture

Understanding the architecture helps you reason about failure modes and scaling.

```text
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Application │     │  Application │     │  Node         │
│  /metrics    │     │  /metrics    │     │  Exporter     │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └───────────┬────────┘────────────────────┘
                   │ scrape (pull)
            ┌──────▼───────┐
            │  Prometheus  │
            │  Server      │──── AlertManager ──── PagerDuty/Slack
            │  (TSDB)      │
            └──────┬───────┘
                   │ query
            ┌──────▼───────┐
            │   Grafana    │
            └──────────────┘
```

Key characteristics:

- **Pull-based:** Prometheus scrapes `/metrics` endpoints at configurable intervals (typically 15–30 seconds)
- **Local storage:** TSDB stores data on local disk with configurable retention (default 15 days)
- **Remote write:** For long-term storage, Prometheus can forward data to Thanos, Cortex, Mimir, or cloud backends

### Scrape Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'my-service'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['my-service:8080']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

## Recording Rules

When a PromQL query is expensive and used frequently (e.g., in dashboards and alerts), pre-compute it with a recording rule:

```yaml
# rules/recording.yml
groups:
  - name: http_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))

      - record: job:http_error_ratio:rate5m
        expr: job:http_errors:rate5m / job:http_requests:rate5m
```

Recording rules run at the configured interval and store the result as a new time series, making dashboard queries instant.

## Key Takeaways

- Four metric types exist: counters (cumulative), gauges (current value), histograms (bucketed distributions), and summaries (client-side quantiles) — prefer histograms for latency measurement
- Label cardinality is the primary scaling concern: never use unbounded values (user IDs, request IDs) as metric labels
- PromQL's `rate()` function is the foundation for counter-based queries; always apply rate before aggregating counters
- Follow naming conventions strictly: snake_case, base units as suffix (`_seconds`, `_bytes`), `_total` for counters
- Recording rules pre-compute expensive queries and dramatically improve dashboard performance
- Prometheus is pull-based — applications expose a `/metrics` endpoint and Prometheus scrapes it at configurable intervals
