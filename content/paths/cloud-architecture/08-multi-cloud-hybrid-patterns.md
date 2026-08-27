---
title: "Multi-Cloud & Hybrid Patterns"
weight: 8
---

# Multi-Cloud & Hybrid Patterns

Multi-cloud is one of the most over-sold and under-scrutinised strategies in cloud architecture. Vendors and analysts promote it as best practice; practitioners know it doubles operational complexity. This section separates the legitimate use cases from the cargo-cult adoption and covers the patterns that make multi-cloud and hybrid architectures work when they're genuinely needed.

---

## When Multi-Cloud Makes Sense

### Legitimate Reasons

| Reason | Scenario | Example |
|--------|----------|---------|
| **Best-of-breed services** | A specific provider has a uniquely superior service | BigQuery for analytics + AWS for everything else |
| **M&A inheritance** | Acquisition brings workloads on a different cloud | Company A (AWS) acquires Company B (Azure) |
| **Regulatory requirement** | Regulator mandates no single-vendor dependency | Financial services in some jurisdictions |
| **Data sovereignty** | Provider lacks a region in a required country | GCP has no region in a required market; Azure does |
| **Negotiation leverage** | Credible multi-cloud capability improves pricing negotiations | Demonstrably portable workloads |

### Bad Reasons (But Commonly Cited)

| Claimed Reason | Reality |
|----------------|---------|
| "Avoid vendor lock-in" | You trade one lock-in (cloud) for another (abstraction layer). The abstraction layer is your own code, which is harder to maintain than managed services. |
| "Redundancy across clouds" | Cross-cloud failover is extraordinarily complex. Within-cloud multi-region is far simpler and better tested. |
| "We want flexibility" | Flexibility without usage is waste. Design for portability only when you'll actually port. |
| "Our team knows both" | Your team will become mediocre at two clouds instead of excellent at one. |

---

## The Cost of Multi-Cloud

Multi-cloud is not free. Quantify these costs before committing:

```
Multi-Cloud Overhead
│
├── Operational Complexity
│   ├── Two sets of IAM models
│   ├── Two networking architectures
│   ├── Two monitoring/alerting systems
│   ├── Two CI/CD pipelines
│   └── Two incident response playbooks
│
├── Engineering Cost
│   ├── Abstraction layers to maintain
│   ├── Lowest-common-denominator design (can't use best features)
│   ├── Doubled learning curve for every engineer
│   └── More complex testing matrices
│
├── Financial Cost
│   ├── Cross-cloud data transfer (egress fees)
│   ├── Lost volume discounts (spend split across providers)
│   ├── Duplicate tooling licences
│   └── Additional support contracts
│
└── Reliability Risk
    ├── More failure modes (cross-cloud connectivity)
    ├── Harder to debug distributed issues
    └── Blast radius now spans two clouds
```

---

## Vendor Lock-In Analysis

Lock-in is real, but it's a spectrum, not a binary. Evaluate it per service, not per cloud.

### Lock-In Risk by Service Category

| Category | Lock-In Risk | Portability | Migration Effort |
|----------|-------------|------------|-----------------|
| Compute (VMs) | Low | Docker images run anywhere | Days |
| Containers (K8s) | Low | K8s is portable; managed add-ons aren't | Weeks |
| Object storage | Low | S3 API is the de facto standard | Days–weeks |
| Relational DB (PostgreSQL) | Low | Standard SQL, multiple managed options | Weeks |
| Serverless functions | Medium | Code is portable; triggers and IAM aren't | Weeks–months |
| Proprietary DB (DynamoDB, Cosmos) | High | Data model is provider-specific | Months |
| ML/AI services | High | Models are portable; training pipelines aren't | Months |
| Identity (IAM) | Very High | Every provider's IAM model is unique | Months |
| Networking (VPC, TGW) | Very High | Architecture is provider-specific | Months |
| Workflow/orchestration | High | Step Functions, Workflows are not interchangeable | Months |

### Lock-In Decision Framework

```
For each service:
│
├── Is there a portable standard? (SQL, K8s, S3 API, OCI)
│   ├── Yes → Use it. Portability is built in.
│   └── No
│       ├── Is there a credible open-source alternative?
│       │   ├── Yes → Evaluate: is the managed service worth the lock-in?
│       │   │         (Usually yes — operational savings outweigh switching cost)
│       │   └── No → Accept the lock-in, but isolate the dependency behind an interface
│       └── Is this a strategic or commodity service?
│           ├── Strategic → Accept lock-in for competitive advantage (e.g., Spanner for global consistency)
│           └── Commodity → Prefer the portable option
```

---

## Abstraction Layers

If you pursue multi-cloud, you need abstraction layers to shield application code from provider-specific APIs. But abstractions have costs.

### Abstraction Approaches

| Approach | Complexity | Portability | Performance |
|----------|-----------|-------------|-------------|
| **Cloud-agnostic IaC** (Terraform, Pulumi) | Medium | Infrastructure only | N/A |
| **Kubernetes everywhere** | High | Compute + networking | Near-native |
| **Application-level interfaces** | Medium | Per-service | Slight overhead |
| **Service mesh** (Istio, Consul) | High | Networking + observability | Overhead |
| **Portable ML frameworks** (MLflow, Kubeflow) | Medium | ML pipelines | Slight overhead |

### Cloud-Agnostic IaC

Terraform is the most common tool for multi-cloud IaC. Each provider has its own Terraform provider, but the workflow (plan → apply → state) is consistent.

```
┌─────────────────────────────────────────┐
│              Terraform Code              │
│                                          │
│  module "aws_vpc" {                      │
│    source = "./modules/aws/networking"   │
│  }                                       │
│                                          │
│  module "azure_vnet" {                   │
│    source = "./modules/azure/networking" │
│  }                                       │
│                                          │
│  module "gcp_vpc" {                      │
│    source = "./modules/gcp/networking"   │
│  }                                       │
└─────────────────────────────────────────┘
        │           │           │
        ▼           ▼           ▼
   AWS API     Azure API    GCP API
```

**Limitation:** Terraform abstracts the *workflow*, not the *resources*. An `aws_vpc` and an `azurerm_virtual_network` are different resources with different arguments. You still need provider-specific knowledge.

---

## Cross-Cloud Data Patterns

Data synchronisation across clouds is one of the hardest multi-cloud problems.

### Data Sync Patterns

| Pattern | Latency | Consistency | Complexity |
|---------|---------|------------|-----------|
| **ETL/batch sync** | Hours | Eventual (batch window) | Low |
| **CDC (Change Data Capture)** | Seconds–minutes | Eventual (stream lag) | Medium |
| **Dual-write** | Real-time | Requires distributed TX or saga | Very high |
| **Shared object storage** | Minutes | Eventual | Low |
| **API-based sync** | Real-time | Request-level | Medium |

### Cross-Cloud Architecture

```
┌────────────── AWS ──────────────┐    ┌────────── GCP ──────────────┐
│                                  │    │                              │
│  ┌──────────┐   ┌────────────┐  │    │  ┌────────────┐             │
│  │ App Tier  │──►│ PostgreSQL │  │    │  │ BigQuery   │             │
│  │ (ECS)    │   │ (RDS)      │  │    │  │ (Analytics)│             │
│  └──────────┘   └─────┬──────┘  │    │  └─────▲──────┘             │
│                        │         │    │        │                     │
│                        │ CDC     │    │        │ Load               │
│                        ▼         │    │        │                     │
│               ┌────────────────┐ │    │  ┌─────┴──────┐             │
│               │ Kafka / Debez- │─┼────┼─►│ Dataflow   │             │
│               │ ium (CDC)      │ │    │  │ (ingest)   │             │
│               └────────────────┘ │    │  └────────────┘             │
│                                  │    │                              │
└──────────────────────────────────┘    └──────────────────────────────┘
                   Cross-cloud data: CDC from AWS → GCP for analytics
```

### Egress Cost Awareness

| Transfer | AWS | GCP | Azure |
|----------|-----|-----|-------|
| Same region, same AZ | Free | Free | Free |
| Same region, cross-AZ | $0.01/GB | Free | Free |
| Cross-region | $0.02/GB | $0.01–0.08/GB | $0.02/GB |
| To internet (egress) | $0.09/GB (first 10TB) | $0.12/GB | $0.087/GB |
| To another cloud | Same as internet egress | Same | Same |

**Cross-cloud egress is billed as internet egress — the most expensive tier.** Budget for it or minimise it.

---

## Hybrid Architecture Patterns

Hybrid means some workloads on-prem, some in the cloud. It's the most common state during migration and sometimes the permanent state for regulated industries.

### Hybrid Patterns

| Pattern | Description | Use Case |
|---------|------------|----------|
| **Cloud bursting** | Baseline on-prem, burst to cloud for peaks | Seasonal retail, batch processing |
| **Tiered hybrid** | Dev/test in cloud, prod on-prem (or reverse) | Migration phases, cost optimisation |
| **Edge + cloud** | Edge devices process locally, sync to cloud | IoT, retail POS, factory floors |
| **Data hybrid** | Regulated data on-prem, analytics in cloud | Healthcare, financial services |
| **Disaster recovery** | Primary on-prem, DR in cloud | Cost-effective DR without second DC |

### Consistency Across Environments

| Challenge | Solution |
|-----------|---------|
| Different IAM models | Federate identity from a single IdP |
| Different networking | Site-to-site VPN or Direct Connect with consistent DNS |
| Different monitoring | Centralised observability platform (Datadog, Grafana Cloud) |
| Different IaC | Terraform for both (if cloud on-prem tools like vSphere have providers) |
| Different CI/CD | Single pipeline tool deploying to multiple targets |

---

## Service Mesh Across Clouds

A service mesh provides consistent networking, security, and observability across services regardless of where they run.

### Cross-Cloud Mesh Architecture

```
┌──────────── AWS ────────────────┐    ┌──────────── GCP ──────────────┐
│                                  │    │                                │
│  ┌─────┐  ┌─────┐  ┌─────┐     │    │  ┌─────┐  ┌─────┐            │
│  │Svc A│  │Svc B│  │Svc C│     │    │  │Svc D│  │Svc E│            │
│  │+proxy│  │+proxy│  │+proxy│     │    │  │+proxy│  │+proxy│            │
│  └──┬──┘  └──┬──┘  └──┬──┘     │    │  └──┬──┘  └──┬──┘            │
│     │        │        │         │    │     │        │                │
│  ───┴────────┴────────┴───      │    │  ───┴────────┴───             │
│        Mesh Data Plane          │    │    Mesh Data Plane            │
│                                  │    │                                │
│     Control Plane (local)        │    │   Control Plane (local)       │
└────────────────┬─────────────────┘    └──────────┬─────────────────────┘
                  │         Federation             │
                  └──────────────┬─────────────────┘
                                 │
                          Global Control
                          (Consul, Istio multi-cluster)
```

**When this is worth the complexity:** You have services that must communicate across clouds with mTLS, traffic management, and unified observability. For most organisations, this is unnecessary — pick one cloud and use its native service mesh.

---

## Key Takeaways

- Multi-cloud doubles operational complexity — pursue it only when there's a specific, measurable benefit (best-of-breed, M&A, regulation)
- "Avoiding vendor lock-in" is rarely a sufficient justification — the abstraction layer becomes its own maintenance burden
- Evaluate lock-in risk per service, not per cloud; use portable standards (SQL, K8s, S3 API) where they exist
- Cross-cloud data transfer is billed at internet egress rates — factor this into every multi-cloud cost model
- Terraform provides workflow abstraction but not resource abstraction — you still need provider-specific expertise
- Hybrid architecture is the pragmatic reality for most migrations; invest in consistent identity, networking, and observability across environments
