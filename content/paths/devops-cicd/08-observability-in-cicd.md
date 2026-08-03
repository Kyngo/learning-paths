---
title: "Observability in CI/CD"
weight: 8
---

# Observability in CI/CD

You can't improve what you don't measure. Observability applied to CI/CD pipelines means tracking DORA metrics, understanding where time is spent, catching regressions early, and verifying deployments automatically.

---

## DORA Metrics

The four key metrics from the DevOps Research and Assessment (DORA) team that predict software delivery performance:

| Metric | Definition | Elite | High | Medium | Low |
|--------|-----------|-------|------|--------|-----|
| **Deployment Frequency** | How often code reaches production | On-demand (multiple/day) | Weekly to monthly | Monthly to 6-monthly | Fewer than once per 6 months |
| **Lead Time for Changes** | Commit to production | < 1 hour | 1 day to 1 week | 1 week to 1 month | > 6 months |
| **Change Failure Rate** | % of deployments causing failure | 0-15% | 16-30% | 16-30% | > 30% |
| **Mean Time to Recovery (MTTR)** | Time to restore service after failure | < 1 hour | < 1 day | 1 day to 1 week | > 6 months |

### Measuring Deployment Frequency

```yaml
# GitLab CI: emit deployment event to Datadog
deploy_prod:
  stage: deploy
  script:
    - ./deploy.sh
    - |
      curl -X POST "https://api.datadoghq.eu/api/v1/events" \
        -H "DD-API-KEY: ${DD_API_KEY}" \
        -d @- <<EOF
      {
        "title": "Production Deployment",
        "text": "Deployed ${CI_PROJECT_NAME} ${CI_COMMIT_TAG}",
        "tags": ["service:${CI_PROJECT_NAME}", "env:prod", "team:${TEAM}"],
        "alert_type": "success"
      }
      EOF
```

### Measuring Lead Time

Lead time = time from first commit on branch to production deployment:

```text
Commit pushed → PR opened → Review → Merge → Build → Test → Deploy (prod)
|←─────────────────── Lead Time ──────────────────────────────────────→|
```

Track by recording:
1. First commit timestamp (from Git)
2. Production deployment timestamp (from deploy job)
3. Delta = lead time

### Measuring Change Failure Rate

```text
CFR = (failed deployments / total deployments) × 100

"Failed" = deployment that causes:
  - Rollback
  - Hotfix within 24h
  - Incident created
  - Service degradation alert
```

### Measuring MTTR

```text
MTTR = mean(time_service_restored - time_incident_detected)
```

Sources: incident management system (PagerDuty, Opsgenie), monitoring alerts, deployment events.

---

## Pipeline Metrics

Beyond DORA, track the health of your pipelines themselves:

| Metric | What It Tells You | Target |
|--------|------------------|--------|
| Pipeline duration | Total wall time from trigger to finish | < 10 min (CI), < 30 min (full CD) |
| Queue time | Time waiting for a runner | < 30 seconds |
| Flaky test rate | Tests that pass/fail non-deterministically | < 1% |
| Build success rate | % of pipeline runs that succeed | > 95% |
| Stage duration breakdown | Where time is spent (build, test, deploy) | Identify bottlenecks |
| Pipeline frequency | How often pipelines run | Tracks team velocity |
| Cache hit rate | How often build caches are reused | > 80% |

### Tracking Pipeline Duration Over Time

```yaml
# GitHub Actions: output timing to summary
- name: Record pipeline metrics
  if: always()
  run: |
    START=${{ github.event.workflow_run.created_at }}
    END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    echo "## Pipeline Metrics" >> $GITHUB_STEP_SUMMARY
    echo "- **Duration:** calculated from job timestamps" >> $GITHUB_STEP_SUMMARY
    echo "- **Status:** ${{ job.status }}" >> $GITHUB_STEP_SUMMARY
    echo "- **Commit:** ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
```

---

## Build Notifications

Notify the right people at the right time, without alert fatigue:

### Notification Strategy

| Event | Channel | Who | Urgency |
|-------|---------|-----|---------|
| Pipeline failed on main | Slack team channel | Whole team | High — blocks everyone |
| Pipeline failed on feature branch | Author DM or PR comment | Committer only | Medium — self-serve fix |
| Deploy to production succeeded | Slack deploy channel | Stakeholders | Info |
| Deploy to production failed | Slack + PagerDuty | On-call engineer | Critical |
| Flaky test detected | Slack bot + issue created | Test owner | Low — track for cleanup |

### Slack Notification Example (GitLab CI)

```yaml
notify_failure:
  stage: .post
  script:
    - |
      curl -X POST "$SLACK_WEBHOOK_URL" \
        -H 'Content-Type: application/json' \
        -d "{
          \"text\": \"❌ Pipeline failed on *${CI_PROJECT_NAME}* (${CI_COMMIT_REF_NAME})\",
          \"blocks\": [{
            \"type\": \"section\",
            \"text\": {
              \"type\": \"mrkdwn\",
              \"text\": \"*Pipeline:* <${CI_PIPELINE_URL}|#${CI_PIPELINE_ID}>\n*Branch:* ${CI_COMMIT_REF_NAME}\n*Author:* ${GITLAB_USER_NAME}\"
            }
          }]
        }"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_failure
```

---

## Pipeline Dashboards

Visualise pipeline health for the team:

### What to Display

```text
┌─────────────────────────────────────────────────────┐
│  DORA Metrics (last 30 days)                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐│
│  │ Deploy   │ │ Lead     │ │ CFR      │ │ MTTR   ││
│  │ Freq: 4/d│ │ Time: 2h │ │ 8%       │ │ 22min  ││
│  └──────────┘ └──────────┘ └──────────┘ └────────┘│
├─────────────────────────────────────────────────────┤
│  Pipeline Health                                     │
│  Success rate: 94% ████████████████░░ (7d)          │
│  Avg duration: 8m 23s (trend: ↓12%)                │
│  Flaky tests:  3 (down from 7 last week)            │
├─────────────────────────────────────────────────────┤
│  Recent Deployments                                  │
│  ✅ v1.4.2 → prod (12 min ago)                     │
│  ✅ v1.4.1 → prod (2 days ago)                     │
│  ❌ v1.4.1-rc1 → pre (rolled back, 3 days ago)     │
└─────────────────────────────────────────────────────┘
```

### Tools for Pipeline Dashboards

| Tool | Source Data | Best For |
|------|------------|----------|
| Grafana + Prometheus | CI system metrics, custom exporters | Custom dashboards, full control |
| Datadog CI Visibility | GitLab/GitHub/Jenkins natively | Enterprise, correlated with APM |
| GitLab Value Stream Analytics | GitLab native | GitLab-only shops |
| Sleuth | Git + deploy + incident data | DORA tracking as a service |
| LinearB | Git + project management | Engineering metrics platform |

---

## Post-Deploy Verification

Deployment is not done when the container is running — verify it's working correctly:

### Verification Layers

| Layer | What It Checks | When | Automated? |
|-------|---------------|------|-----------|
| Health check | Container responds on `/health` | Immediately | Yes (orchestrator) |
| Smoke tests | Critical paths return 200 | First 2 minutes | Yes (pipeline job) |
| Synthetic monitoring | Full user journeys | Continuously | Yes (Datadog/Checkly) |
| Canary analysis | Error rate/latency vs baseline | First 10-30 min | Yes (Argo Rollouts, Flagger) |
| Manual verification | Visual/UX inspection | After auto-checks pass | No |

### Automated Smoke Test in Pipeline

```yaml
post_deploy_verify:
  stage: verify
  script:
    - |
      echo "Waiting for service to be healthy..."
      for i in $(seq 1 30); do
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.prod.example.com/health)
        if [ "$STATUS" = "200" ]; then
          echo "✅ Health check passed"
          break
        fi
        sleep 10
      done
      if [ "$STATUS" != "200" ]; then
        echo "❌ Health check failed after 5 minutes"
        exit 1
      fi
    - |
      echo "Running smoke tests..."
      curl -sf https://api.prod.example.com/v1/status | jq .version
      curl -sf https://api.prod.example.com/v1/ping
  after_script:
    - |
      if [ "$CI_JOB_STATUS" = "failed" ]; then
        echo "⚠️ Triggering rollback..."
        ./scripts/rollback.sh
      fi
```

### Automatic Rollback Criteria

| Signal | Threshold | Action |
|--------|-----------|--------|
| 5xx error rate | > 5% for 2 min | Rollback |
| P99 latency | > 2x baseline for 5 min | Rollback |
| Health endpoint | Down for 1 min | Rollback |
| Exception rate | > 3x baseline | Alert + manual decision |

### Canary Deployment Verification

```text
Deploy canary (10% traffic)
         │
         ▼
Monitor for 10 minutes
  ├── Error rate within threshold? ──NO──▶ Rollback
  ├── Latency within threshold?   ──NO──▶ Rollback
  └── All good? ──────────────────────────▶ Promote to 100%
```

---

## Continuous Improvement Loop

Use observability data to improve your delivery process:

```text
Measure → Identify bottleneck → Experiment → Measure again

Examples:
- Pipeline slow? → Parallelise test stages, add caching
- High CFR? → Add more pre-deploy tests, shrink batch size
- Long lead time? → Reduce review bottlenecks, trunk-based development
- Frequent rollbacks? → Better canary analysis, feature flags
```

---

## Key Takeaways

1. **Track DORA metrics** — they're the industry standard for measuring delivery performance. Start with deployment frequency and lead time.
2. **Pipeline duration is developer experience** — a slow pipeline kills feedback loops. Target under 10 minutes for CI.
3. **Notify smartly** — failed main = team alert, failed feature branch = author only. Avoid alert fatigue.
4. **Post-deploy verification is mandatory** — health checks, smoke tests, and canary analysis catch issues before users do.
5. **Automatic rollback** on clear failure signals (5xx spike, health check failure) reduces MTTR to near-zero for detected issues.
6. **Dashboard your metrics** — visibility drives improvement. If the team can't see pipeline health at a glance, they won't improve it.
