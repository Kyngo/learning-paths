---
title: "Cost Architecture"
weight: 10
---

# Cost Architecture

Cost is an architecture concern, not a finance concern. Every design decision — the database engine, the compute model, the replication strategy — has a cost implication. Engineers who understand cost architecture build systems that are both performant and economically sustainable. This section covers the principles, patterns, and organisational structures for cost-aware cloud design.

---

## Cost-Aware Architecture Principles

### The Five Principles

| Principle | What It Means |
|-----------|--------------|
| **Cost is a first-class requirement** | Treat cost like latency or availability — it has targets, is measured, and is reviewed |
| **Right-size, don't over-provision** | Match resources to actual usage, not theoretical peak |
| **Pay for what you use** | Prefer per-request/per-second pricing when workloads are variable |
| **Design for cost visibility** | Every resource is tagged, every cost is attributable to a team/service |
| **Review continuously** | Cost optimisation is not a one-time project; it's a recurring practice |

### Cost as a Non-Functional Requirement

Just as you specify SLOs for latency and availability, specify cost targets:

```
Service: Booking API
├── Availability SLO: 99.95%
├── Latency P99: < 200ms
├── Cost target: < £0.50 per 1,000 bookings
└── Cost ceiling: £8,000/month at current volume
```

---

## Architecture Trade-Offs with Cost Impact

Every architecture decision has cost implications. The table below maps common decisions to their cost profile.

### Compute Trade-Offs

| Decision | Lower Cost | Higher Cost | When to Accept Higher |
|----------|-----------|-------------|----------------------|
| Serverless vs containers | Serverless at low traffic | Containers at low traffic | Serverless wins below ~5M requests/month |
| Spot vs on-demand | Spot (60–90% cheaper) | On-demand | On-demand when interruption is unacceptable |
| ARM vs x86 | ARM (20–40% cheaper) | x86 | x86 only for binary compatibility |
| Right-sized instances | Smaller, matched to usage | Over-provisioned "just in case" | Over-provision temporarily for launches |

### Data Trade-Offs

| Decision | Lower Cost | Higher Cost | When to Accept Higher |
|----------|-----------|-------------|----------------------|
| Single-region vs multi-region | Single-region | Multi-region (replication + egress) | Multi-region for global latency or DR |
| Storage tiering | Cold/archive for old data | All data on hot storage | Hot when all data is frequently accessed |
| Managed DB vs self-managed | Self-managed (lower per-unit) | Managed (higher per-unit) | Managed to save engineering time |
| Read replicas | Single instance | Multiple read replicas | Replicas when read scaling is needed |

### Networking Trade-Offs

| Decision | Lower Cost | Higher Cost | Notes |
|----------|-----------|-------------|-------|
| Same-AZ deployment | Free data transfer | Reduced availability | Acceptable for dev, not for prod |
| NAT Gateway vs NAT Instance | NAT Instance | NAT Gateway (~£30/mo + data) | NAT Gateway for reliability |
| Private Link vs public endpoint | Private Link (avoid egress) | Public (egress charges) | Private Link also improves security |
| VPC endpoints | Free (gateway endpoints for S3/DynamoDB) | Paid (interface endpoints) | Always enable gateway endpoints |

---

## Reserved Capacity Strategies

Reserved capacity (Reserved Instances, Savings Plans, Committed Use Discounts) trades flexibility for lower unit cost.

### Commitment Comparison

| Model | Discount | Flexibility | Commitment |
|-------|----------|------------|-----------|
| On-demand | 0% | Full | None |
| Savings Plan (AWS) / CUD (GCP) | 30–60% | Medium (any instance family/region) | 1 or 3 years |
| Reserved Instance (specific) | 40–72% | Low (specific type/region) | 1 or 3 years |
| Spot/preemptible | 60–90% | High (but interruptible) | None |

### Commitment Strategy

```
Total compute spend
│
├── Baseline (always-on, predictable)
│   └── 70% covered by Savings Plans / CUD (1-year)
│
├── Steady growth
│   └── 20% covered by Savings Plans (flexible)
│
└── Variable/burst
    └── 10% on-demand + spot
```

### When NOT to Commit

| Situation | Why |
|-----------|-----|
| Workload may be retired in < 12 months | You'll pay for unused capacity |
| Instance type is likely to change | Graviton migration, architecture change |
| Traffic is highly unpredictable | You'll over-commit or under-commit |
| Organisation is early in cloud adoption | Usage patterns aren't established |

---

## Spot Architecture Patterns

Spot instances are the most powerful cost lever in cloud computing. Designing for spot requires embracing interruption as a feature, not a failure.

### Spot-Friendly Workloads

| Workload | Spot-Friendly? | Why |
|----------|---------------|-----|
| CI/CD runners | ✅ | Short-lived, retryable |
| Batch processing | ✅ | Checkpointable, queue-driven |
| Data pipelines (Spark, ETL) | ✅ | Retryable stages, data in object storage |
| Dev/test environments | ✅ | Interruption is acceptable |
| Stateless web tier (with on-demand base) | ✅ | Mixed fleet: OD base + spot burst |
| Databases | ❌ | Stateful, interruption causes data risk |
| Real-time trading / payments | ❌ | Latency-sensitive, interruption unacceptable |

### Spot Savings Architecture

```
Without Spot:
┌──────────────────────────────────────┐
│  10x m6i.xlarge (on-demand)          │
│  = 10 × £0.192/hr = £1.92/hr        │
│  = £1,382/month                      │
└──────────────────────────────────────┘

With Mixed Fleet (30% OD + 70% Spot):
┌──────────────────────────────────────┐
│  3x m6i.xlarge (on-demand)           │
│  = 3 × £0.192/hr = £0.576/hr        │
│                                      │
│  7x m6i.xlarge (spot, ~70% discount) │
│  = 7 × £0.058/hr = £0.406/hr        │
│                                      │
│  Total = £0.982/hr                   │
│  = £707/month (49% savings)          │
└──────────────────────────────────────┘
```

---

## Cost Allocation Models

### Allocation Approaches

| Model | How It Works | Best For |
|-------|-------------|----------|
| **Direct attribution** | Each resource tagged to a team/service; cost = usage | IaaS, per-service databases |
| **Proportional split** | Shared resources divided by usage metric (requests, CPU) | Shared clusters, shared networking |
| **Fixed allocation** | Flat fee per team/service regardless of usage | Simplicity, platform teams |
| **Showback** | Teams see their cost but don't pay | Early-stage cost awareness |
| **Chargeback** | Teams pay for their usage from their budget | Mature organisations, accountability |

### Tagging for Cost Allocation

```
Every resource must have:
├── Team: platform / bookings / search / payments
├── Environment: dev / pre / prod
├── CostCentre: CC-1234
├── Application: booking-api / search-indexer
└── ManagedBy: terraform / manual

Cost Explorer / Billing Dashboard
├── Filter by Team=bookings, Environment=prod
│   └── Shows: £12,430/month
│       ├── EC2: £5,200
│       ├── RDS: £3,800
│       ├── S3: £430
│       └── Data Transfer: £3,000
└── Filter by CostCentre=CC-1234
    └── Shows: £45,000/month (entire department)
```

---

## FinOps Team Structure

FinOps (Financial Operations) is the practice of bringing financial accountability to cloud spend. It's a cross-functional discipline, not a finance task.

### FinOps Team Model

```
┌─────────────────────────────────────────────────────────────┐
│                     FinOps Practice                           │
│                                                               │
│  ┌─────────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ Engineering      │  │ Finance     │  │ Leadership     │  │
│  │                 │  │             │  │                │  │
│  │ • Cost tools    │  │ • Budgets   │  │ • Priorities   │  │
│  │ • Architecture  │  │ • Forecasts │  │ • Trade-offs   │  │
│  │ • Optimisation  │  │ • Reporting │  │ • Investment   │  │
│  │ • Tagging       │  │ • Variance  │  │   decisions    │  │
│  └─────────────────┘  └─────────────┘  └────────────────┘  │
│                                                               │
│  Cadence: Weekly cost review, Monthly deep-dive,             │
│           Quarterly commitment planning                       │
└─────────────────────────────────────────────────────────────┘
```

### FinOps Maturity Phases

| Phase | Focus | Activities |
|-------|-------|-----------|
| **Crawl** | Visibility | Enable cost allocation, tagging, basic dashboards |
| **Walk** | Optimisation | Right-sizing, reserved capacity, spot adoption, waste elimination |
| **Run** | Operations | Unit economics, automated recommendations, engineering culture change |

---

## Unit Economics

Unit economics measure cost efficiency in business terms, not infrastructure terms. They answer: "how much does it cost to serve one customer / process one transaction / store one GB?"

### Defining Unit Metrics

| Business | Unit Metric | Target |
|----------|------------|--------|
| E-commerce | Cost per order | £0.12 per order |
| SaaS platform | Cost per active user per month | £1.50 per user/month |
| Data pipeline | Cost per GB processed | £0.03 per GB |
| API service | Cost per 1,000 API calls | £0.08 per 1K calls |
| Media platform | Cost per hour of video streamed | £0.002 per hour |

### Unit Economics Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│                   Cost Per Booking                            │
│                                                               │
│  £0.55 ┤                                                     │
│        │     ╱╲                                              │
│  £0.45 ┤    ╱  ╲        Target: £0.50                       │
│        │   ╱    ╲─────────────────────── £0.42               │
│  £0.35 ┤  ╱                                                  │
│        │ ╱                                                   │
│  £0.25 ┤╱                                                    │
│        ├─────┬─────┬─────┬─────┬─────┬─────┬─────           │
│        Jan   Feb   Mar   Apr   May   Jun   Jul               │
│                                                               │
│  Trend: Improving (right-sizing + Graviton migration)        │
└─────────────────────────────────────────────────────────────┘
```

### Why Unit Economics Matter

| Metric | What It Tells You |
|--------|------------------|
| Total cloud spend | How much you're spending (but not if it's efficient) |
| Cost per service | Which services are expensive (but not if they're worth it) |
| **Cost per business unit** | Whether cloud spend scales efficiently with business growth |

**The goal:** As the business grows, total spend increases but **cost per unit decreases** (economies of scale). If cost per unit increases, your architecture doesn't scale economically.

---

## Cost Optimisation Quick Wins

| Action | Effort | Typical Savings | First Step |
|--------|--------|----------------|-----------|
| Delete unused resources | Low | 5–15% | List unattached EBS volumes, unused EIPs, idle LBs |
| Right-size instances | Low | 10–30% | Review Compute Optimizer / recommendations |
| Savings Plans / CUD | Low | 20–40% | Analyse 30-day usage, commit for baseline |
| Storage tiering | Low | 20–60% on storage | Enable lifecycle policies on S3 / GCS |
| Spot instances | Medium | 40–70% on batch/CI | Start with CI/CD runners, expand to batch |
| Graviton / ARM migration | Medium | 20% on compute | Test on non-production, migrate compatible workloads |
| NAT Gateway optimisation | Low | 10–30% on networking | Route S3/DynamoDB through VPC gateway endpoints |
| Data transfer reduction | Medium | Variable | Compress payloads, cache at edge, use Private Link |

---

## Key Takeaways

- Treat cost as a first-class architecture requirement alongside latency, availability, and security
- Every architecture decision has a cost implication — evaluate compute, data, and networking trade-offs explicitly
- Cover baseline compute with reserved capacity (Savings Plans, CUD); use spot for burst and batch workloads
- Tag every resource for cost allocation; without tags, you cannot attribute, optimise, or budget
- FinOps is cross-functional: engineering provides optimisation, finance provides budgets, leadership provides priorities
- Measure unit economics (cost per transaction, per user, per GB) — total spend is meaningless without business context
- Cost optimisation is continuous, not a one-time project; review weekly, deep-dive monthly, plan commitments quarterly
