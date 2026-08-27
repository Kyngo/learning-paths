---
title: "Introduction to Observability"
weight: 1
---

# Introduction to Observability

Observability is the measure of how well you can understand a system's internal state from its external outputs. The term originates from control theory, where a system is "observable" if its internal state can be reconstructed from its outputs alone. In software engineering, observability means your production systems emit enough telemetry — metrics, logs, and traces — that you can diagnose any failure without deploying new code.

## Observability vs Monitoring

These terms are related but distinct. Understanding the difference is essential.

**Monitoring** answers predefined questions: "Is the server up?", "Is CPU above 80%?", "Did the deploy succeed?" You decide what to check in advance, set thresholds, and get alerted when conditions breach them.

**Observability** lets you ask *new* questions you didn't anticipate: "Why are requests from users in Germany slower than usual?", "Which downstream service is causing the spike in P99 latency?", "What changed between yesterday's successful deployments and today's failure?"

| Aspect | Monitoring | Observability |
|--------|-----------|---------------|
| Question type | Known unknowns (predefined checks) | Unknown unknowns (ad-hoc investigation) |
| Data model | Aggregated metrics, fixed dashboards | High-cardinality, high-dimensionality data |
| Response to failure | Alert fires → check runbook | Explore telemetry → form hypothesis → verify |
| Typical tooling | Nagios, Zabbix, CloudWatch Alarms | Datadog, Honeycomb, Grafana + Tempo + Loki |
| Mindset | "Is it broken?" | "Why is it broken? What else is affected?" |

### When Monitoring Falls Short

Consider a microservices architecture with 50 services. A monitoring setup might track:

- HTTP 5xx rate per service
- CPU and memory per host
- Database connection pool usage

This catches known failure modes. But when a user reports "checkout is slow for orders containing more than 10 items", monitoring cannot answer that question — it lacks the dimensions (order size, user geography, item type) to slice the data.

Observability systems retain high-cardinality dimensions so you can group-by, filter, and correlate in real time.

## The Three Pillars

The three pillars of observability are the fundamental telemetry signal types:

### Metrics

Metrics are numeric measurements aggregated over time. They tell you *what* is happening at a system level.

```text
http_requests_total{method="POST", endpoint="/api/orders", status="500"} 142
```

**Strengths:** Low storage cost, fast queries, excellent for dashboards and alerts, long retention.

**Limitations:** Pre-aggregated — you lose individual request detail. Adding dimensions after the fact requires re-instrumentation.

### Logs

Logs are timestamped, immutable records of discrete events. They tell you *what happened* in a specific context.

```json
{
  "timestamp": "2026-08-27T08:15:32.004Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc123def456",
  "message": "Payment gateway timeout",
  "order_id": "ORD-9182",
  "duration_ms": 30042,
  "gateway": "stripe"
}
```

**Strengths:** Rich context, human-readable, flexible schema, useful for audit trails.

**Limitations:** Expensive to store at scale, slow to query without indexing, noisy without structure.

### Traces

Traces follow a single request as it flows through multiple services. They tell you *where* time is spent and *how* services interact.

```text
Trace abc123def456
├── [200ms] api-gateway: POST /api/orders
│   ├── [5ms]  auth-service: validate-token
│   ├── [45ms] order-service: create-order
│   │   ├── [12ms] inventory-service: check-stock
│   │   └── [30s]  payment-service: charge ← TIMEOUT
│   └── [2ms]  notification-service: send-confirmation (not reached)
```

**Strengths:** End-to-end visibility, pinpoints bottlenecks, reveals service dependencies.

**Limitations:** High data volume, sampling required at scale, complex to implement across all services.

### How the Pillars Complement Each Other

No single pillar is sufficient. They work together:

1. **Metrics** alert you that the error rate spiked at 14:32
2. **Logs** from that time window show `PaymentGatewayTimeoutException` errors
3. **Traces** reveal that `payment-service` calls to Stripe are timing out at 30 seconds, but only for orders containing more than 10 line items

| Signal | Tells you | Cardinality | Cost | Query speed |
|--------|-----------|-------------|------|-------------|
| Metrics | What is happening (aggregate) | Low–medium | Low | Fast (sub-second) |
| Logs | What happened (event detail) | High | High | Medium (seconds) |
| Traces | How a request flowed | Very high | Very high | Medium (seconds) |

## Beyond Three Pillars

Modern observability extends the three pillars with additional signals:

### Profiles

Continuous profiling captures CPU, memory, and I/O usage at the function level. Tools like Pyroscope, Datadog Continuous Profiler, and Parca attach profiling data to traces, showing *why* a span is slow — not just *that* it's slow.

### Events

Structured events (deploys, config changes, feature flag toggles) are overlaid on dashboards to correlate system changes with behaviour shifts. If latency increased at 14:32 and a deploy happened at 14:30, the correlation is immediately visible.

### Real User Monitoring (RUM)

RUM captures telemetry from end-user browsers and mobile apps — page load times, JavaScript errors, Core Web Vitals — and links them back to server-side traces.

## Why Observability Matters

### Reduced MTTR

Mean Time to Recovery (MTTR) is the primary metric observability improves. Teams with mature observability resolve incidents in minutes rather than hours because they can:

- Identify the failing component within seconds
- Understand the blast radius (which users, which endpoints, which regions)
- Pinpoint the root cause without guesswork

### Confidence in Change

Observability gives deployment confidence. When you can see the real-time impact of a canary deployment — error rate, latency percentiles, business metrics — you can roll back in seconds if something goes wrong.

### System Understanding

Complex distributed systems are impossible to fully model mentally. Observability makes implicit behaviours explicit: hidden dependencies, unexpected retry storms, cascading timeouts.

## Key Concepts

### Cardinality

Cardinality is the number of unique combinations of label values for a metric. High cardinality is powerful (you can slice data by user ID, request ID, etc.) but expensive to store and query.

```text
Low cardinality:    http_requests{method="GET|POST|PUT|DELETE"}           → 4 series
Medium cardinality: http_requests{method="..", endpoint="/api/..."}       → ~100 series
High cardinality:   http_requests{method="..", endpoint="..", user_id="."} → millions of series
```

Metrics systems (Prometheus) struggle with high cardinality. Tracing and logging systems handle it natively.

### Dimensionality

Dimensionality is the number of label *keys* (fields) attached to telemetry. More dimensions mean more ways to slice data, but also more storage and indexing cost.

### Telemetry Pipeline

Raw telemetry flows through a pipeline before it becomes useful:

```text
Application → Agent/SDK → Collector → Backend → Query/Dashboard
             (emit)      (collect)   (process)  (store)    (visualise)
```

The OpenTelemetry project standardises the first three stages (emit, collect, process), while backends and visualisation tools vary by platform.

## The Observability Maturity Model

Teams typically progress through stages:

| Level | Characteristics | Typical tools |
|-------|----------------|---------------|
| **0 — None** | No systematic telemetry; SSH into boxes and tail logs | `ssh`, `tail -f` |
| **1 — Reactive** | Basic health checks and uptime monitoring | Nagios, Pingdom, CloudWatch Alarms |
| **2 — Proactive** | Dashboards, structured logs, some alerting on SLIs | Grafana, ELK, PagerDuty |
| **3 — Observability** | Correlated metrics/logs/traces, SLO-based alerting, runbooks | Datadog, Grafana Cloud, Honeycomb |
| **4 — Advanced** | Continuous profiling, eBPF, AIOps, automated remediation | Parca, Cilium, custom ML pipelines |

## Getting Started Checklist

Before instrumenting anything, establish these foundations:

1. **Define service ownership** — Every service has an owner responsible for its telemetry
2. **Standardise naming** — Agree on metric names, log fields, and span attributes across teams
3. **Choose a backend** — Pick a platform (or self-hosted stack) and commit to it
4. **Instrument the golden signals first** — Latency, traffic, errors, saturation for every service
5. **Connect the pillars** — Ensure logs and traces share a `trace_id` so you can jump between views

## Key Takeaways

- Observability is about asking new questions of your systems; monitoring is about checking known conditions — you need both, but observability scales better for complex architectures
- The three pillars — metrics, logs, and traces — complement each other: metrics for alerting, logs for context, traces for request flow
- High cardinality is the distinguishing feature of observability; it allows slicing data by arbitrary dimensions but increases cost
- Modern observability extends beyond three pillars to include profiles, events, and real user monitoring
- MTTR is the primary business metric improved by observability investment — faster diagnosis means faster recovery
- Start with the golden signals (latency, traffic, errors, saturation) and correlate across pillars using shared trace IDs
