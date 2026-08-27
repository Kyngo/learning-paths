---
title: "Observability"
weight: 83
bookCollapseSection: true
description: "Understand and implement observability across metrics, logs, and traces — from instrumentation to incident response."
---

# Observability

Observability is the ability to understand a system's internal state from its external outputs. Unlike traditional monitoring — which checks known failure modes — observability lets you ask arbitrary questions about your production systems without deploying new code. This path covers the entire observability stack: from raw signals to actionable insights.

## What You'll Learn

This path takes you from foundational concepts through production-grade observability:

1. **Introduction to Observability** — The three pillars, observability vs monitoring, and why it matters for modern distributed systems
2. **Metrics** — Counter, gauge, histogram, and summary types; Prometheus data model; PromQL fundamentals
3. **Logging** — Structured logging, log levels, JSON format, aggregation pipelines, ELK/EFK, CloudWatch Logs
4. **Distributed Tracing** — OpenTelemetry, spans, trace context propagation, Jaeger, X-Ray, Zipkin
5. **Instrumentation** — OpenTelemetry SDK, auto vs manual instrumentation, language-specific examples
6. **Dashboards & Visualisation** — Grafana, dashboard design, RED method, USE method, golden signals
7. **Alerting** — SLOs, SLIs, error budgets, burn rate alerts, alert fatigue, on-call integration
8. **Observability Platforms** — Datadog, Grafana Cloud, New Relic, AWS native tools, cost comparison
9. **Observability in Practice** — Debugging production issues, runbooks, on-call best practices, incident response
10. **Advanced Topics** — Continuous profiling, eBPF, synthetic monitoring, chaos engineering, AIOps

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Introduction to Observability]({{< relref "01-introduction" >}}) | Three pillars, observability vs monitoring, MTTR, cardinality, telemetry signals |
| 2 | [Metrics]({{< relref "02-metrics" >}}) | Counter, gauge, histogram, summary, Prometheus data model, PromQL, naming conventions |
| 3 | [Logging]({{< relref "03-logging" >}}) | Structured logging, log levels, JSON logs, ELK/EFK stack, CloudWatch Logs, log aggregation |
| 4 | [Distributed Tracing]({{< relref "04-distributed-tracing" >}}) | OpenTelemetry, spans, trace context, W3C Trace Context, Jaeger, X-Ray, Zipkin |
| 5 | [Instrumentation]({{< relref "05-instrumentation" >}}) | OpenTelemetry SDK, auto vs manual instrumentation, Java, Python, Go examples |
| 6 | [Dashboards & Visualisation]({{< relref "06-dashboards-and-visualisation" >}}) | Grafana, RED method, USE method, golden signals, dashboard design principles |
| 7 | [Alerting]({{< relref "07-alerting" >}}) | SLOs, SLIs, SLAs, error budgets, burn rate alerts, alert fatigue, PagerDuty, OpsGenie |
| 8 | [Observability Platforms]({{< relref "08-observability-platforms" >}}) | Datadog, Grafana Cloud, New Relic, CloudWatch, X-Ray, cost comparison |
| 9 | [Observability in Practice]({{< relref "09-observability-in-practice" >}}) | Debugging production issues, runbooks, on-call best practices, incident response |
| 10 | [Advanced Topics]({{< relref "10-advanced-topics" >}}) | Continuous profiling, eBPF, synthetic monitoring, chaos engineering, AIOps |

## Prerequisites

- Familiarity with HTTP, REST APIs, and web services
- Basic Linux command-line skills
- Understanding of microservices or distributed architectures
- Some experience deploying or operating applications (any cloud or on-premises)

## Who This Is For

- Backend and platform engineers building or maintaining production services
- SREs and DevOps engineers responsible for reliability and incident response
- Team leads establishing observability standards across services
- Anyone transitioning from "we have monitoring" to "we have observability"

## How to Use This Path

Start with the Introduction to understand the conceptual framework, then work through Metrics, Logging, and Distributed Tracing as the three foundational pillars. Instrumentation teaches you how to emit those signals from code. Dashboards, Alerting, and Platforms show how to consume and act on them. The final two sections cover real-world practice and advanced techniques.

**Recommended time:** 2–3 hours per section, plus hands-on practice with Prometheus, Grafana, and OpenTelemetry.
