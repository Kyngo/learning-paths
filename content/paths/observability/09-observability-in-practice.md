---
title: "Observability in Practice"
weight: 9
---

# Observability in Practice

Observability tools are only as valuable as the processes around them. This section covers the human side: how to debug production issues systematically, write effective runbooks, structure on-call rotations, and run incident response. These practices turn raw telemetry into faster recovery and fewer repeat incidents.

## Debugging Production Issues

### The Systematic Approach

When an alert fires or a user reports an issue, resist the urge to SSH into a server and start guessing. Follow a structured debugging flow:

```text
1. Assess impact     → Who is affected? How many users? Which regions?
2. Identify signal   → Which SLI is breached? Errors? Latency? Both?
3. Narrow scope      → Which service? Which endpoint? Which dependency?
4. Correlate         → What changed? Deploy? Config? Traffic spike?
5. Form hypothesis   → "Payment service is timing out due to DB connection exhaustion"
6. Verify            → Check traces/logs/metrics for evidence
7. Remediate         → Fix or mitigate (rollback, scale, toggle feature flag)
8. Confirm recovery  → Verify SLIs return to normal
```

### Debugging with the Three Pillars

**Step 1 — Metrics (what is happening):**

```promql
# Check error rate spike
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)

# Check latency
histogram_quantile(0.99, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))

# Check saturation
max by (service) (container_memory_usage_bytes / container_spec_memory_limit_bytes)
```

The dashboard shows: `payment-service` error rate jumped from 0.1% to 12% at 14:32.

**Step 2 — Traces (where is the problem):**

Find traces from `payment-service` with errors:

```text
{ resource.service.name = "payment-service" && status = error }
```

Traces show: spans calling the `orders` PostgreSQL database are timing out after 30 seconds. The `pg_pool_acquire` span takes 29.8 seconds.

**Step 3 — Logs (why is it happening):**

```logql
{service="payment-service"} | json | level="ERROR" | line_format "{{.message}}"
```

Logs show: `"Cannot acquire connection from pool — all 20 connections in use"`.

**Root cause:** A slow query introduced in the 14:30 deploy is holding database connections for 10+ seconds, exhausting the connection pool.

**Remediation:** Roll back the deploy, then investigate the slow query.

### Common Production Failure Patterns

| Pattern | Symptoms | Investigation Path |
|---------|----------|-------------------|
| Cascading failure | Service A fails, then B, then C | Trace dependency chain; check circuit breakers |
| Connection pool exhaustion | Timeouts, "cannot acquire connection" | Check pool metrics, slow query logs |
| Memory leak | Gradually increasing memory, eventual OOM | Continuous profiler, heap dumps |
| Retry storm | Traffic amplification, downstream overload | Check retry counts in traces, add backoff |
| DNS resolution failure | Intermittent connection failures | Check DNS metrics, TTL, resolver health |
| Certificate expiry | TLS handshake failures | Check cert expiry dates, automate renewal |
| Clock skew | Trace spans out of order, auth failures | Compare `node_time_seconds` across hosts |
| Noisy neighbour | Intermittent latency spikes | Check host-level metrics (CPU steal, I/O wait) |

## Runbooks

A runbook is a documented procedure for responding to a specific alert or incident type. Every alert that pages a human must link to a runbook.

### Runbook Structure

```markdown
# Runbook: SLOBurnRateCritical

## Alert Description
High error burn rate — consuming error budget at 14.4x the sustainable rate.
This means the monthly error budget will be exhausted in ~2 days if unchecked.

## Impact
Users are experiencing elevated error rates on HTTP requests.

## Dashboard
https://grafana.internal/d/order-service-red

## Diagnosis Steps

### 1. Identify the affected service
Check which service has the highest error rate:
```
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

### 2. Check recent deployments
Look at deploy annotations on the dashboard.
If a deploy happened in the last 30 minutes, consider rolling back.

### 3. Check downstream dependencies
Examine traces for the failing service:
- Are downstream calls timing out?
- Is a database connection pool exhausted?
- Is an external API returning errors?

### 4. Check resource saturation
- CPU: > 80% sustained?
- Memory: > 90%?
- DB connections: near pool limit?

## Remediation

### If caused by a recent deploy
1. Identify the deploy: `glab api projects/<ID>/deployments --per-page=5`
2. Roll back: `kubectl rollout undo deployment/<service-name>`
3. Verify recovery: error rate drops within 5 minutes

### If caused by downstream dependency
1. Check dependency health dashboard
2. If external: enable circuit breaker / fallback
3. If internal: escalate to the owning team

### If caused by resource exhaustion
1. Scale horizontally: increase replica count
2. Investigate the root cause (memory leak, slow queries)

## Escalation
If unresolved after 30 minutes, escalate to the platform team lead.

## Post-Resolution
- Verify SLIs return to normal
- Create an incident report
- Schedule a post-mortem if the impact exceeded 15 minutes
```

### Runbook Best Practices

| Practice | Why |
|----------|-----|
| Link from the alert annotation | On-call engineer clicks directly from PagerDuty to the runbook |
| Include actual commands and queries | Don't make the reader construct PromQL under pressure |
| Keep it up to date | Stale runbooks are worse than none — they waste time and build false confidence |
| Include escalation paths | The on-call engineer must know when and who to escalate to |
| Test runbooks periodically | Run through them during game days |

## On-Call Best Practices

### Rotation Structure

| Aspect | Recommendation |
|--------|---------------|
| Rotation length | 1 week (handoff on weekday mornings, not Fridays) |
| Shift coverage | Primary + secondary on-call for redundancy |
| Team size | Minimum 5 people per rotation to avoid burnout |
| Handoff | Written handoff document summarising active issues and upcoming risks |
| Compensation | Follow local regulations; provide time-off or pay for out-of-hours pages |

### On-Call Responsibilities

The on-call engineer is responsible for:

1. **Acknowledging pages** within 5 minutes
2. **Triaging** the alert (real incident vs false positive)
3. **Following the runbook** to diagnose and remediate
4. **Escalating** if unable to resolve within 30 minutes
5. **Communicating** status updates in the incident channel
6. **Handing off** active issues at rotation end

### Reducing On-Call Burden

| Action | Impact |
|--------|--------|
| Quarterly alert audits | Remove non-actionable alerts |
| Improve runbooks after every incident | Next occurrence is faster to resolve |
| Automate common remediations | Auto-restart, auto-scale, auto-rollback |
| Invest in SLO-based alerting | Fewer, higher-quality alerts |
| Track pages-per-shift metric | Objective measure of on-call health |

## Incident Response

### Incident Severity

| Severity | Criteria | Response |
|----------|----------|----------|
| SEV-1 | Complete outage or data loss | All hands, war room, exec communication |
| SEV-2 | Major feature degraded, significant user impact | On-call + team lead, status page update |
| SEV-3 | Minor feature degraded, limited impact | On-call handles, team notified |
| SEV-4 | Cosmetic issue or internal tooling | Ticket created, handled in normal workflow |

### Incident Lifecycle

```text
Detection → Triage → Response → Remediation → Recovery → Post-Mortem
```

**Detection:** Alert fires, user report, or monitoring anomaly detected.

**Triage (first 5 minutes):**
- Assess severity and blast radius
- Open an incident channel (e.g., `#inc-2026-08-27-payment-timeout`)
- Assign an Incident Commander (IC)

**Response (5–60 minutes):**
- IC coordinates investigation and communication
- Engineers follow runbooks, check dashboards, examine traces
- Status updates every 15 minutes to stakeholders

**Remediation:**
- Apply fix (rollback, config change, scaling, hotfix)
- Verify SLIs return to normal
- Monitor for regression

**Recovery:**
- Confirm all systems nominal for 30+ minutes
- Close the incident channel
- Schedule post-mortem

### Post-Mortems

Post-mortems are blameless retrospectives focused on systemic improvements.

```markdown
# Post-Mortem: Payment Service Timeout (2026-08-27)

## Summary
Payment service connection pool exhausted due to a slow query
introduced in deploy v2.14.3. Checkout errors peaked at 12%
for 23 minutes.

## Timeline
- 14:30 — Deploy v2.14.3 rolled out to production
- 14:32 — Error rate SLO burn rate alert fires
- 14:35 — On-call acknowledges, opens incident channel
- 14:38 — Traces show DB connection pool exhaustion
- 14:42 — Logs confirm slow query on `orders` table
- 14:45 — Rollback to v2.14.2 initiated
- 14:48 — Rollback complete, error rate dropping
- 14:55 — SLIs confirmed normal, incident closed

## Root Cause
A new query in `OrderRepository.findByUserIdAndStatus()` performed
a sequential scan instead of using the composite index. Under load,
each query held a connection for 10+ seconds, exhausting the pool.

## Impact
- Duration: 23 minutes
- Users affected: ~3,200 checkout attempts failed
- Error budget consumed: 4.2% of monthly budget

## What Went Well
- Burn rate alert detected the issue within 2 minutes
- Traces clearly showed the DB bottleneck
- Rollback was executed in under 10 minutes

## What Went Wrong
- No query performance test in CI for new database queries
- Connection pool exhaustion had no dedicated alert
- Deploy happened at 14:30 (peak traffic) instead of low-traffic window

## Action Items
| # | Action | Owner | Due |
|---|--------|-------|-----|
| 1 | Add query explain plan check to CI pipeline | @backend-lead | 2026-09-05 |
| 2 | Add connection pool utilisation alert (>80%) | @sre-team | 2026-09-01 |
| 3 | Define deploy windows (avoid 14:00–16:00) | @team-lead | 2026-09-01 |
| 4 | Add composite index for the affected query | @backend-dev | 2026-08-28 |
```

### Post-Mortem Principles

1. **Blameless** — Focus on systems, not individuals
2. **Evidence-based** — Timeline backed by telemetry (timestamps from alerts, traces, logs)
3. **Action-oriented** — Every post-mortem produces specific, assigned, dated action items
4. **Shared** — Published to the team (or organisation) so others learn
5. **Followed up** — Action items are tracked to completion

## Key Takeaways

- Debug production issues systematically: metrics identify *what* is wrong, traces show *where* the bottleneck is, logs explain *why* it's happening — follow this sequence rather than guessing
- Every paging alert must link to a runbook containing actual commands, queries, and decision trees — on-call engineers should not need to construct PromQL under incident pressure
- On-call rotations need minimum 5 people, weekly handoffs, written handoff documents, and a target of fewer than 2 pages per shift to be sustainable
- Incident response follows a lifecycle: detection → triage → response → remediation → recovery → post-mortem — assign an Incident Commander to coordinate each incident
- Post-mortems must be blameless, evidence-based, and produce tracked action items — without action items, post-mortems are just storytelling
- The most valuable post-mortem output is systemic improvement (better alerts, automated checks, deploy guardrails) not individual blame
