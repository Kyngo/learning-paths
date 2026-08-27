---
title: "Cost Management"
weight: 15
---

# Cost Management

AWS bills can grow faster than your infrastructure if unmanaged. Cost optimisation is a continuous practice — not a one-time cleanup. This section covers the tools, patterns, and organisational practices for keeping AWS spend under control.

---

## Understanding Your Bill

### AWS Cost Structure

| Dimension | What You Pay For |
|-----------|-----------------|
| Compute | EC2 hours, Lambda invocations, Fargate vCPU/memory-hours |
| Storage | S3 GB-months, EBS GB-months, snapshots |
| Data transfer | Outbound to internet, cross-AZ, cross-region |
| Requests | S3 GET/PUT, API Gateway calls, SQS messages |
| Provisioned capacity | RDS instances (running 24/7), NAT Gateways, Elasticsearch |

### Common Cost Surprises

| Surprise | Cause | Fix |
|----------|-------|-----|
| NAT Gateway data fees | All private subnet outbound traffic goes through NAT | Use VPC endpoints for AWS services, minimise cross-AZ |
| Idle RDS instances | Dev/staging databases running 24/7 | Stop/start on schedule, or use Aurora Serverless v2 |
| Unattached EBS volumes | Volumes from terminated EC2 instances | Automated cleanup |
| Old snapshots | Years of daily EBS/RDS snapshots | Lifecycle policies |
| Cross-AZ data transfer | $0.01/GB between AZs adds up | Keep chatty services in the same AZ |
| CloudWatch Logs | High log volume without retention | Set retention policies (7/14/30 days) |

---

## AWS Cost Tools

### Cost Explorer

Visualise and analyse spend over time:

- Filter by service, account, tag, region, instance type
- Group by any dimension
- Forecast future spend based on trends
- Identify cost anomalies

### AWS Budgets

Set spending thresholds and get alerts:

```
Budget: $5,000/month for production account
Alert 1: 80% threshold ($4,000) → email team
Alert 2: 100% threshold ($5,000) → email + SNS → Lambda auto-response
Alert 3: Forecast exceeds budget → early warning
```

### Cost and Usage Report (CUR)

The most detailed billing data — line-item level, exported to S3. Use with Athena for custom analysis:

```sql
SELECT line_item_product_code, SUM(line_item_unblended_cost) as cost
FROM cur_data
WHERE month = '2024-01'
GROUP BY line_item_product_code
ORDER BY cost DESC
LIMIT 10;
```

---

## Pricing Models

### On-Demand vs Reserved vs Savings Plans vs Spot

| Model | Discount | Commitment | Best For |
|-------|----------|-----------|----------|
| On-Demand | 0% | None | Variable workloads, testing |
| Reserved Instances | 30-72% | 1 or 3 year, specific instance type | Steady-state production |
| Savings Plans | 30-72% | 1 or 3 year, $/hour commitment | Flexible (any instance family) |
| Spot Instances | 60-90% | None (can be terminated with 2 min warning) | Batch processing, CI/CD, fault-tolerant |

### Savings Plans vs Reserved Instances

| | Savings Plans | Reserved Instances |
|-|---------------|-------------------|
| Flexibility | Any instance family, any region | Specific instance type and AZ |
| Applies to | EC2, Fargate, Lambda | EC2 only (or RDS/ElastiCache separately) |
| Recommendation | **Prefer for most cases** | Only if you need capacity reservation |

---

## Right-Sizing

Match instance sizes to actual utilisation:

```
Current: m5.2xlarge (8 vCPU, 32 GB) — avg CPU 15%, avg memory 20%
Right-sized: m5.large (2 vCPU, 8 GB) — saves ~75%
```

### Tools

- **AWS Compute Optimizer** — analyses CloudWatch metrics, recommends instance types
- **Cost Explorer Right-Sizing Recommendations** — identifies underutilised instances
- **Trusted Advisor** — checks for idle resources, underutilised instances

### Auto-Scaling

Don't run peak capacity 24/7:

```
Baseline: 2 instances (handles normal traffic)
Peak: 10 instances (auto-scales for traffic spikes)
Schedule: scale down to 1 instance at night (non-production)
```

---

## Tagging Strategy

Tags are the foundation of cost allocation:

| Tag | Purpose | Example |
|-----|---------|---------|
| `Environment` | Filter by env | `production`, `staging`, `development` |
| `Team` | Cost allocation to team | `platform`, `frontend`, `data` |
| `Project` | Cost allocation to project | `checkout`, `search`, `ml-pipeline` |
| `CostCenter` | Finance mapping | `CC-1234` |
| `ManagedBy` | Who manages the resource | `terraform`, `manual`, `cdk` |

**Enforce tags with AWS Organizations SCPs or AWS Config rules.** Untagged resources are unaccountable.

---

## FinOps Practices

### The FinOps Loop

```
Inform → Optimise → Operate → Repeat
```

1. **Inform:** Visibility into who spends what (tags, CUR, dashboards)
2. **Optimise:** Right-size, purchase commitments, eliminate waste
3. **Operate:** Continuous governance (budgets, alerts, anomaly detection)

### Quick Wins Checklist

- [ ] Delete unattached EBS volumes and old snapshots
- [ ] Set CloudWatch Logs retention (not "Never expire")
- [ ] Stop dev/staging RDS instances outside business hours
- [ ] Use S3 Intelligent-Tiering for infrequently accessed data
- [ ] Enable S3 Lifecycle rules (move to Glacier after 90 days)
- [ ] Use VPC endpoints instead of NAT Gateway for AWS service traffic
- [ ] Review and terminate unused Elastic IPs, load balancers, NAT gateways
- [ ] Buy Savings Plans for steady-state compute
- [ ] Enable Cost Anomaly Detection in Cost Explorer
- [ ] Tag every resource — enforce via SCP or Config rule

---

## Key Takeaways

- Cost surprises come from NAT Gateway data fees, idle databases, old snapshots, and CloudWatch Logs retention.
- Savings Plans offer 30-72% discounts with more flexibility than Reserved Instances. Prefer them for compute.
- Right-sizing is the highest-impact single action — most instances run at 10-20% utilisation.
- Tags are mandatory for cost accountability. Untagged resources are invisible to cost analysis.
- FinOps is a continuous loop: inform, optimise, operate. Set budgets and alerts from day one.
- Use Cost Explorer for analysis, Budgets for alerts, Compute Optimizer for right-sizing, and CUR for deep dives.
