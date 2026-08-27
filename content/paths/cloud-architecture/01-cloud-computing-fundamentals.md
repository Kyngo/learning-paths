---
title: "Cloud Computing Fundamentals"
weight: 1
---

# Cloud Computing Fundamentals

Cloud computing is renting someone else's computers — but the architecture decisions you make on those rented computers are entirely different from on-premises. This section covers the service models, shared responsibilities, and design principles that underpin every cloud-native system.

---

## Cloud Service Models

Every cloud provider offers services across a spectrum of abstraction. Understanding where each model sits determines what you manage versus what the provider manages.

```
┌─────────────────────────────────────────────────────────────────┐
│                    YOU MANAGE EVERYTHING                         │
│   On-Premises                                                    │
│   ┌──────────┬──────────┬──────────┬──────────┬──────────┐      │
│   │ App      │ App      │ App      │ App      │ App      │      │
│   │ Runtime  │ Runtime  │ Runtime  │ Runtime  │ Runtime  │      │
│   │ OS       │ OS       │ OS       │ OS       │ OS       │      │
│   │ Compute  │ Compute  │ Compute  │ Compute  │ Compute  │      │
│   │ Storage  │ Storage  │ Storage  │ Storage  │ Storage  │      │
│   │ Network  │ Network  │ Network  │ Network  │ Network  │      │
│   └──────────┴──────────┴──────────┴──────────┴──────────┘      │
├─────────────────────────────────────────────────────────────────┤
│   IaaS            PaaS           FaaS           SaaS            │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│   │ App    ▲ │   │ App    ▲ │   │ Code   ▲ │   │        ▲ │   │
│   │ Runtime│ │   │        │ │   │        │ │   │        │ │   │
│   │ OS     │Y│   │ Runtime│P│   │ Runtime│P│   │ App    │P│   │
│   │ ───────│O│   │ OS     │R│   │ OS     │R│   │ Runtime│R│   │
│   │ Compute│U│   │ ───────│O│   │ ───────│O│   │ OS     │O│   │
│   │ Storage│ │   │ Compute│V│   │ Compute│V│   │ Compute│V│   │
│   │ Network│ │   │ Storage│ │   │ Storage│ │   │ Storage│ │   │
│   └────────┘ │   │ Network│ │   │ Network│ │   │ Network│ │   │
│              │   └────────┘ │   └────────┘ │   └────────┘ │   │
│   YOU ▲      │              │              │              │   │
│   PROVIDER ─ │              │              │              │   │
└─────────────────────────────────────────────────────────────────┘
```

### Service Model Comparison

| Model | You Manage | Provider Manages | Examples |
|-------|-----------|-----------------|----------|
| **IaaS** | OS, runtime, app, data | Hardware, networking, virtualisation | EC2, Compute Engine, Azure VMs |
| **PaaS** | App code, data | OS, runtime, scaling, patching | App Engine, Elastic Beanstalk, Azure App Service |
| **FaaS** | Function code | Everything else, including scaling to zero | Lambda, Cloud Functions, Azure Functions |
| **SaaS** | Configuration, data | The entire application | Gmail, Salesforce, Datadog |
| **CaaS** | Container images, orchestration config | Infrastructure, container runtime | ECS, GKE, AKS |
| **DBaaS** | Schema, queries, data | Engine, patching, backups, scaling | RDS, Cloud SQL, Cosmos DB |

### Choosing a Service Model

```
Is your team managing the OS and patching today?
├── Yes, and we want to keep control → IaaS
├── Yes, but we'd rather not → PaaS or CaaS
└── We just want to ship code
    ├── Stateless, event-driven → FaaS
    ├── Long-running processes → PaaS or CaaS
    └── Standard business app → SaaS (buy, don't build)
```

---

## The Shared Responsibility Model

Security and operations responsibilities split between you and the provider. The split point varies by service model.

| Responsibility | IaaS | PaaS | FaaS | SaaS |
|---------------|------|------|------|------|
| Physical security | Provider | Provider | Provider | Provider |
| Network infrastructure | Provider | Provider | Provider | Provider |
| Hypervisor | Provider | Provider | Provider | Provider |
| Operating system | **You** | Provider | Provider | Provider |
| Runtime/middleware | **You** | Provider | Provider | Provider |
| Application code | **You** | **You** | **You** | Provider |
| Data classification | **You** | **You** | **You** | **You** |
| Identity & access | **You** | **You** | **You** | **You** |
| Client-side encryption | **You** | **You** | **You** | **You** |

**The rule:** The provider secures *the cloud*. You secure what you put *in* the cloud.

Even with SaaS, you are responsible for access control (who can log in), data governance (what data you upload), and compliance (whether using that SaaS meets your regulatory obligations).

---

## Cloud-Native Principles

Cloud-native is not "running on the cloud." It means designing systems that exploit the cloud's unique properties: elasticity, on-demand resources, managed services, and global reach.

### The Five Properties of Cloud-Native Design

1. **Designed for automation** — infrastructure is code, deployments are pipelines, recovery is scripted
2. **Designed for resilience** — failure is expected, systems self-heal, no single points of failure
3. **Designed for elasticity** — resources scale with demand, not with forecasts
4. **Designed for observability** — every component emits metrics, logs, and traces
5. **Designed for loose coupling** — services communicate through well-defined interfaces, not shared state

### Cloud-Native vs Cloud-Hosted

| Characteristic | Cloud-Hosted | Cloud-Native |
|---------------|-------------|-------------|
| Deployment | Lift-and-shift VM | Containers or serverless |
| Scaling | Vertical (bigger machine) | Horizontal (more instances) |
| State | Local disk, session affinity | External stores, stateless processes |
| Configuration | Config files on disk | Environment variables, config services |
| Recovery | Manual restart, runbooks | Auto-healing, rolling deploys |
| Updates | Maintenance windows | Blue-green, canary, zero downtime |

---

## The Twelve-Factor App Methodology

Twelve-factor is a methodology for building applications that deploy cleanly to cloud platforms. Published by Heroku engineers in 2012, it remains the foundation of cloud-native design.

### The Twelve Factors

| Factor | Principle | Cloud Implication |
|--------|----------|------------------|
| **I. Codebase** | One codebase per app, tracked in version control | One repo → one deployable unit |
| **II. Dependencies** | Explicitly declare and isolate dependencies | No reliance on system packages; use lockfiles |
| **III. Config** | Store config in the environment | No hardcoded URLs, credentials, or feature flags |
| **IV. Backing Services** | Treat databases, queues, caches as attached resources | Swap a local PostgreSQL for RDS without code changes |
| **V. Build, Release, Run** | Strictly separate build, release, and run stages | CI builds the artefact, CD releases it, runtime runs it |
| **VI. Processes** | Execute the app as stateless processes | No sticky sessions, no local file state |
| **VII. Port Binding** | Export services via port binding | The app is self-contained; no app server required |
| **VIII. Concurrency** | Scale out via the process model | More instances, not bigger instances |
| **IX. Disposability** | Fast startup, graceful shutdown | Containers start in seconds, handle SIGTERM |
| **X. Dev/Prod Parity** | Keep dev, staging, and production similar | Same container image across all environments |
| **XI. Logs** | Treat logs as event streams | Write to stdout; let the platform aggregate |
| **XII. Admin Processes** | Run admin tasks as one-off processes | Database migrations as CI jobs, not manual SSH |

### Beyond Twelve Factors

Modern cloud-native apps extend the original twelve factors:

| Extension | Principle |
|-----------|----------|
| **API First** | Design the contract before the implementation |
| **Telemetry** | Build in health checks, metrics, and tracing from day one |
| **Authentication/Authorisation** | Treat security as a first-class concern, not a bolt-on |

---

## Cloud vs On-Premises Decision Framework

Not every workload belongs in the cloud. Use this framework to make the decision systematically.

### Decision Criteria

| Criterion | Favours Cloud | Favours On-Prem |
|-----------|-------------|----------------|
| **Demand variability** | Spiky, unpredictable, seasonal | Flat, predictable, 24/7 baseline |
| **Time to market** | Speed matters more than cost optimisation | Long-term cost optimisation is priority |
| **Capital availability** | Prefer OpEx (pay-as-you-go) | CapEx budget already allocated |
| **Data sovereignty** | Provider has regions in required jurisdictions | Strict requirement for physical control |
| **Latency requirements** | Users distributed globally | Users co-located with data centre |
| **Regulatory environment** | Regulations permit cloud (most do) | Regulations require physical control |
| **Team skills** | Team can learn cloud operations | Team expertise is in physical infrastructure |
| **Scale trajectory** | Growing rapidly or uncertain | Stable and well-understood |

### Total Cost of Ownership (TCO)

On-prem TCO is consistently underestimated because it excludes hidden costs:

```
On-Prem TCO
├── Hardware (servers, storage, networking)
├── Software licences (OS, hypervisor, monitoring)
├── Data centre (power, cooling, space, physical security)
├── Personnel (sysadmins, network engineers, security team)
├── Procurement lead time (weeks to months)
├── Over-provisioning (buying for peak, paying at idle)
├── Refresh cycles (hardware EOL every 3–5 years)
└── Opportunity cost (engineers managing hardware, not building product)

Cloud TCO
├── Compute (on-demand, reserved, spot)
├── Storage (per GB, tiered)
├── Network (egress charges, cross-region transfer)
├── Managed services (per request, per hour)
├── Personnel (cloud engineers, fewer but specialised)
└── Operational overhead (monitoring, cost management)
```

### Hybrid Considerations

Most organisations land in a hybrid state: some workloads on-prem, some in the cloud. This is fine if it's intentional — and expensive if it's accidental.

**Good reasons for hybrid:**
- Gradual migration (workloads move in waves)
- Data gravity (large datasets are expensive to move)
- Regulatory requirements (specific data must stay on-prem)
- Latency-sensitive edge processing

**Bad reasons for hybrid:**
- "We're not sure if cloud works" (it does; test with a non-critical workload)
- "Our team isn't ready" (invest in training, not in maintaining two platforms)
- "We already bought the hardware" (sunk cost — evaluate future costs, not past spend)

---

## Cloud Provider Landscape

The three major providers share core concepts but differ in naming and implementation:

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Virtual machine | EC2 | Virtual Machine | Compute Engine |
| Managed Kubernetes | EKS | AKS | GKE |
| Serverless functions | Lambda | Functions | Cloud Functions |
| Object storage | S3 | Blob Storage | Cloud Storage |
| Relational database | RDS | SQL Database | Cloud SQL |
| Virtual network | VPC | VNet | VPC |
| Identity | IAM | Entra ID | Cloud IAM |
| IaC native | CloudFormation | ARM/Bicep | Deployment Manager |
| CLI | `aws` | `az` | `gcloud` |

**Transferable skill:** Learn the *pattern* (object storage, virtual network, managed database), not just the vendor name. The pattern transfers; the CLI flags don't.

---

## Key Takeaways

- Cloud service models (IaaS → SaaS) represent a spectrum of abstraction — the higher you go, the less you manage, but the less control you have
- The shared responsibility model defines the security boundary: the provider secures the infrastructure, you secure your data and access
- Cloud-native means designing for automation, resilience, elasticity, observability, and loose coupling — not just "running on a cloud VM"
- The 12-factor methodology provides concrete guidelines for building cloud-portable applications
- Cloud vs on-prem is a business decision driven by demand variability, time to market, data sovereignty, and total cost of ownership
- Learn patterns, not just vendor-specific services — the principles transfer across AWS, Azure, and GCP
