---
title: "Alerting"
weight: 7
---

# Alerting

Alerting bridges the gap between dashboards and action. A well-designed alerting system notifies the right person, at the right time, with enough context to act — without drowning the team in noise. This section covers SLO-based alerting, error budgets, burn rate alerts, alert design, and on-call integration.

## SLIs, SLOs, and SLAs

### Service Level Indicators (SLIs)

An SLI is a quantitative measure of a specific aspect of service quality. It's the raw measurement:

| SLI | What It Measures | Example |
|-----|-----------------|---------|
| Availability | Proportion of successful requests | 99.95% of requests return non-5xx |
| Latency | Proportion of requests faster than threshold | 99% of requests complete in < 300ms |
| Throughput | Rate of successfully processed work | 10,000 orders/hour |
| Correctness | Proportion of correct results | 99.99% of calculations return accurate results |

### Service Level Objectives (SLOs)

An SLO is a target value for an SLI over a time window. It's the contract your team commits to:

```text
SLO: 99.9% of HTTP requests return non-5xx status over a 30-day rolling window
SLI: (successful requests / total requests) * 100
Target: 99.9%
Window: 30 days rolling
```

**Key insight:** An SLO of 99.9% means you are allowed 0.1% errors — this is your **error budget**.

### Service Level Agreements (SLAs)

An SLA is a contractual obligation with financial consequences (refunds, credits) if the SLO is breached. SLAs are typically weaker than internal SLOs to provide a safety margin.

| Layer | Example | Consequence of Breach |
|-------|---------|----------------------|
| SLA (external) | 99.9% availability | Financial penalty to customer |
| SLO (internal) | 99.95% availability | Engineering priority escalation |
| SLI (measurement) | Currently at 99.97% | Raw data, no consequence |

## Error Budgets

The error budget is the inverse of the SLO — the amount of unreliability you can tolerate:

```text
SLO: 99.9% availability over 30 days
Total minutes in 30 days: 43,200
Error budget: 0.1% × 43,200 = 43.2 minutes of downtime allowed

Alternatively, for request-based:
Total requests in 30 days: 10,000,000
Error budget: 0.1% × 10,000,000 = 10,000 failed requests allowed
```

### Error Budget Policy

The error budget drives engineering decisions:

| Budget Status | Action |
|--------------|--------|
| > 50% remaining | Ship features, take risks, experiment |
| 25–50% remaining | Proceed with caution, prioritise reliability work |
| < 25% remaining | Freeze non-critical changes, focus on reliability |
| Exhausted (0%) | Stop all feature work; only reliability improvements until budget replenishes |

This transforms the "reliability vs velocity" debate from opinion-based to data-driven.

## Alert Design

### What Makes a Good Alert

Every alert must have these properties:

| Property | Description |
|----------|-------------|
| **Actionable** | Someone can and should do something about it *now* |
| **Relevant** | It indicates actual or imminent user impact |
| **Unique** | It doesn't duplicate another alert |
| **Contextual** | It includes enough information to start investigating |
| **Timely** | It fires soon enough to prevent or minimise impact |

### What Makes a Bad Alert

| Symptom | Cause | Fix |
|---------|-------|-----|
| Alert fires but nobody acts | Not actionable | Remove or downgrade to warning |
| Same alert fires 50 times | Missing deduplication or grouping | Group by service, add inhibition |
| Alert fires, resolves, fires, resolves | Flapping threshold | Add `for` duration or hysteresis |
| 3 AM page for a non-urgent issue | Wrong severity | Use severity levels; page only for critical |
| Alert says "CPU high" with no context | Missing runbook or labels | Add runbook link and diagnostic labels |

### Alert Severity Levels

| Level | When to Use | Notification | Response Time |
|-------|------------|-------------|---------------|
| **Critical (P1)** | Users impacted now | Page (PagerDuty/OpsGenie) | Immediate |
| **Warning (P2)** | Degradation or risk of impact | Slack channel | Within 1 hour |
| **Info** | Notable event, no action needed | Dashboard annotation | Next business day |

**Rule:** If an alert pages someone at 3 AM, it must be critical. If it's not worth waking someone up for, it's not a page-worthy alert.

## Burn Rate Alerts

Traditional threshold alerts ("error rate > 1%") are fragile — they don't account for how quickly you're consuming your error budget. Burn rate alerts solve this.

### Concept

The **burn rate** is how fast you're consuming your error budget relative to the SLO window:

```text
Burn rate = (actual error rate) / (SLO-allowed error rate)

If SLO allows 0.1% errors and current error rate is 0.5%:
Burn rate = 0.5 / 0.1 = 5x

At 5x burn rate, a 30-day error budget is exhausted in 6 days (30 / 5).
```

### Multi-Window Burn Rate Alerts

Google SRE recommends alerting on two windows simultaneously — a long window to catch sustained issues and a short window to ensure the problem is still ongoing:

| Alert | Long Window | Short Window | Burn Rate | Budget Consumed | Detects |
|-------|-------------|-------------|-----------|----------------|---------|
| Page (critical) | 1 hour | 5 minutes | 14.4x | 2% in 1h | Fast, complete outage |
| Page (high) | 6 hours | 30 minutes | 6x | 5% in 6h | Significant degradation |
| Ticket (warning) | 3 days | 6 hours | 1x | 10% in 3d | Slow burn |

### Prometheus Alerting Rules

```yaml
# alerting-rules.yml
groups:
  - name: slo-burn-rate
    rules:
      # Error ratio over different windows
      - record: slo:http_error_ratio:rate1h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1h]))
          / sum(rate(http_requests_total[1h]))

      - record: slo:http_error_ratio:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))

      - record: slo:http_error_ratio:rate6h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[6h]))
          / sum(rate(http_requests_total[6h]))

      - record: slo:http_error_ratio:rate30m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[30m]))
          / sum(rate(http_requests_total[30m]))

      # Critical: 14.4x burn rate (2% budget consumed in 1 hour)
      - alert: SLOBurnRateCritical
        expr: |
          slo:http_error_ratio:rate1h > (14.4 * 0.001)
          and
          slo:http_error_ratio:rate5m > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error burn rate — 2% of monthly budget consumed in 1 hour"
          description: "Error rate {{ $value | humanizePercentage }} over 1h window (14.4x burn rate)"
          runbook_url: "https://runbooks.internal/slo-burn-rate"

      # Warning: 6x burn rate (5% budget consumed in 6 hours)
      - alert: SLOBurnRateWarning
        expr: |
          slo:http_error_ratio:rate6h > (6 * 0.001)
          and
          slo:http_error_ratio:rate30m > (6 * 0.001)
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Elevated error burn rate — 5% of monthly budget consumed in 6 hours"
          description: "Error rate {{ $value | humanizePercentage }} over 6h window (6x burn rate)"
          runbook_url: "https://runbooks.internal/slo-burn-rate"
```

### Latency SLO Burn Rate

The same pattern applies to latency SLOs:

```yaml
      # Latency SLO: 99% of requests < 300ms
      - record: slo:http_latency_good_ratio:rate1h
        expr: |
          sum(rate(http_request_duration_seconds_bucket{le="0.3"}[1h]))
          / sum(rate(http_request_duration_seconds_count[1h]))

      - alert: LatencySLOBurnRateCritical
        expr: |
          (1 - slo:http_latency_good_ratio:rate1h) > (14.4 * 0.01)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Latency SLO burn rate critical — P99 breach"
```

## Alertmanager Configuration

Prometheus Alertmanager handles alert routing, grouping, silencing, and notification:

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: 'default-slack'
  group_by: ['alertname', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      group_wait: 10s
      repeat_interval: 1h
    - match:
        severity: warning
      receiver: 'slack-warnings'
      repeat_interval: 4h

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '<PAGERDUTY_INTEGRATION_KEY>'
        description: '{{ .CommonAnnotations.summary }}'
        details:
          service: '{{ .CommonLabels.service }}'
          runbook: '{{ .CommonAnnotations.runbook_url }}'

  - name: 'slack-warnings'
    slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'
        channel: '#alerts-warnings'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'

  - name: 'default-slack'
    slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'
        channel: '#alerts-info'

inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'service']
```

## Alert Fatigue

Alert fatigue occurs when on-call engineers receive so many alerts that they stop paying attention — or worse, start ignoring them.

### Causes and Fixes

| Cause | Indicator | Fix |
|-------|-----------|-----|
| Too many alerts | > 2 pages per on-call shift | Review and remove non-actionable alerts |
| Flapping alerts | Same alert fires/resolves repeatedly | Increase `for` duration, add hysteresis |
| Duplicate alerts | Multiple alerts for the same incident | Use grouping and inhibition rules |
| Non-actionable alerts | Pages with no runbook or unclear action | Every alert must have a runbook |
| Wrong severity | Info-level issues paging at 3 AM | Audit severity; only critical pages |

### Alert Audit Process

Run quarterly:

1. Export all alerts that fired in the last 90 days
2. For each alert: Was it actionable? Did someone act on it? Was the severity appropriate?
3. Delete alerts nobody acted on
4. Downgrade alerts that were always "acknowledged and ignored"
5. Merge duplicate alerts

**Target:** An on-call rotation should produce fewer than 2 pages per 12-hour shift. More than that indicates alert quality problems.

## Key Takeaways

- SLOs define reliability targets; error budgets make the reliability-vs-velocity trade-off data-driven — when the budget is exhausted, feature work stops
- Burn rate alerts are superior to simple threshold alerts because they account for how fast you're consuming your error budget, detecting both fast outages and slow degradation
- Multi-window burn rate alerts use two windows (long + short) to eliminate false positives — the long window detects sustained impact, the short window confirms it's still happening
- Every alert must be actionable, have a severity level, and link to a runbook — if nobody acts on an alert, delete it
- Alert fatigue is the primary failure mode of alerting systems; quarterly audits and a target of fewer than 2 pages per shift keep it under control
- Alertmanager's grouping, inhibition, and routing features are essential for managing alert volume — group by service, inhibit warnings when critical fires, route by severity
