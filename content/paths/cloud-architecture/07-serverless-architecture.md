---
title: "Serverless Architecture"
weight: 7
---

# Serverless Architecture

Serverless doesn't mean "no servers." It means you don't manage, provision, or think about servers. The provider handles scaling, patching, and availability. You write code, define triggers, and pay per execution. This section covers when serverless is the right choice, how to design serverless systems, and the anti-patterns that make serverless expensive or fragile.

---

## Event-Driven Serverless

Serverless functions are inherently event-driven: something happens, a function runs, and it stops. This model maps naturally to many workloads.

### Common Trigger Patterns

| Trigger | Event Source | Use Case |
|---------|-------------|----------|
| HTTP request | API Gateway | REST APIs, webhooks |
| Message | SQS, Pub/Sub, Event Hubs | Async processing, fan-out |
| Schedule | CloudWatch Events, Cloud Scheduler | Cron jobs, periodic tasks |
| File upload | S3, GCS, Blob Storage | Image processing, ETL |
| Database change | DynamoDB Streams, Firestore triggers | Materialised views, audit |
| Stream | Kinesis, Kafka, Event Hubs | Real-time analytics, enrichment |

### Event-Driven Architecture

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐
│  Client   │────►│  API Gateway  │────►│  Function A  │
└──────────┘     └───────────────┘     │  (validate   │
                                        │   & enqueue)  │
                                        └──────┬───────┘
                                               │
                                               ▼
                                        ┌──────────────┐
                                        │  Message      │
                                        │  Queue (SQS)  │
                                        └──────┬───────┘
                                               │
                              ┌────────────────┼────────────────┐
                              ▼                ▼                ▼
                       ┌────────────┐   ┌────────────┐   ┌────────────┐
                       │ Function B │   │ Function C │   │ Function D │
                       │ (process)  │   │ (notify)   │   │ (audit)    │
                       └─────┬──────┘   └─────┬──────┘   └─────┬──────┘
                             │                │                │
                             ▼                ▼                ▼
                       ┌──────────┐    ┌──────────┐    ┌──────────┐
                       │ DynamoDB │    │ SES/SNS  │    │ S3       │
                       └──────────┘    └──────────┘    └──────────┘
```

---

## Cold Starts

A cold start occurs when a function is invoked but no execution environment is available. The provider must allocate resources, load the runtime, and initialise your code.

### Cold Start Breakdown

```
Cold Start Timeline
├── Container allocation       (~50–200ms)    Provider
├── Runtime initialisation     (~50–500ms)    Provider
├── Dependency loading         (~100ms–5s)    Your code
├── Handler initialisation     (~10–100ms)    Your code
└── Function execution         (varies)       Your code
    Total cold start: 200ms – 10s+
```

### Cold Start Factors

| Factor | Impact | Mitigation |
|--------|--------|-----------|
| **Runtime** | JVM/C# cold start > Python/Node/Go | Use Go or Node for latency-sensitive functions |
| **Package size** | Larger package = slower load | Minimise dependencies, use layers/extensions |
| **Memory allocation** | More memory = faster CPU = faster cold start | Allocate 512MB+ even if you don't need the memory |
| **VPC attachment** | ENI creation adds 1–10s | Use VPC only when you need private resources |
| **Provisioned concurrency** | Pre-warms N instances | Use for critical paths (adds cost) |

### Cold Start Decision Framework

```
Is cold start latency acceptable for this workload?
│
├── Not latency-sensitive (async, batch, scheduled)
│   └── Accept cold starts — they're free and rare
│
├── Moderately sensitive (API behind 1s SLO)
│   ├── Use a lightweight runtime (Go, Node, Python)
│   ├── Minimise dependencies
│   └── Keep functions warm with scheduled pings
│
└── Highly sensitive (sub-100ms required)
    ├── Use provisioned concurrency
    ├── Or use containers instead (always warm)
    └── Or put the function behind a cache
```

---

## Function Composition

Individual functions are small. Real systems chain multiple functions together. The challenge is orchestrating them without creating a distributed spaghetti monster.

### Composition Patterns

| Pattern | How It Works | Use Case |
|---------|-------------|----------|
| **Sequential pipeline** | Function A → Queue → Function B → Queue → Function C | ETL, multi-step processing |
| **Fan-out/fan-in** | One event triggers N parallel functions; results aggregated | Batch processing, map-reduce |
| **Orchestration** | A workflow engine coordinates functions | Complex business processes |
| **Choreography** | Functions react to events independently | Loosely coupled domains |

### Orchestration vs Choreography

```
Orchestration (Step Functions / Workflows)
┌─────────────────────────────────────┐
│          Workflow Engine             │
│                                     │
│  Start → Validate → Process → Notify│
│            │                        │
│            ├── On error → Retry     │
│            └── On failure → Alert   │
└─────────────────────────────────────┘
Central control, explicit error handling, visible state

Choreography (Event-driven)
┌──────────┐   Event A   ┌──────────┐   Event B   ┌──────────┐
│ Service 1 │ ──────────► │ Service 2 │ ──────────► │ Service 3 │
└──────────┘             └──────────┘             └──────────┘
                                                       │ Event C
                                                       ▼
                                                 ┌──────────┐
                                                 │ Service 4 │
                                                 └──────────┘
Decentralised, loosely coupled, harder to debug
```

### Workflow Engines

| Service | Provider | Features |
|---------|----------|----------|
| Step Functions | AWS | Visual workflow, Express (high-throughput) and Standard modes |
| Workflows | GCP | YAML-based, HTTP call support, long-running |
| Durable Functions | Azure | Code-based orchestration in C#/Python/JS |
| Temporal | Self-hosted / Cloud | Language-native, replay-based, cross-cloud |

---

## Serverless Databases

Serverless extends beyond compute. Several database services now offer pay-per-query or auto-scaling models.

| Database | Model | Best For |
|----------|-------|----------|
| DynamoDB (on-demand) | Pay per read/write unit | Unpredictable traffic, key-value |
| Aurora Serverless v2 | Scales ACUs (Aurora Capacity Units) | Variable SQL workloads |
| Firestore | Pay per document operation | Mobile/web backends, real-time sync |
| BigQuery | Pay per TB scanned | Ad-hoc analytics, data warehousing |
| Cosmos DB (serverless) | Pay per RU consumed | Global distribution, multi-model |
| Neon / PlanetScale | Scale to zero PostgreSQL/MySQL | Dev environments, low-traffic apps |

### When to Use Serverless Databases

| Good Fit | Poor Fit |
|----------|----------|
| Traffic is unpredictable or spiky | Sustained high throughput (provisioned is cheaper) |
| Application has idle periods | Always-on OLTP with consistent load |
| Development and staging environments | Sub-millisecond latency requirements |
| Prototype or MVP | Complex joins across large datasets |

---

## Serverless Anti-Patterns

### The Distributed Monolith

Deploying a monolith as dozens of Lambda functions that call each other synchronously.

```
BAD: Lambda A → calls → Lambda B → calls → Lambda C → calls → Lambda D
     (4x cold starts, 4x latency, impossible to debug)

GOOD: Lambda A handles the full request
      OR: Use a workflow engine to orchestrate steps
      OR: Use a container if the logic is tightly coupled
```

### The Lambda Pinball

Events bouncing between functions with no clear flow, making the system impossible to trace or debug.

**Fix:** Use a workflow engine (Step Functions, Temporal) for multi-step processes. Reserve choreography for genuinely independent, loosely coupled domains.

### Ignoring Concurrency Limits

Every function has a concurrency limit (default: 1,000 on AWS). A traffic spike can exhaust the limit and throttle your entire application.

**Fix:** Set reserved concurrency for critical functions. Use a queue to buffer requests and smooth out spikes.

### Over-granular Functions

One function per API endpoint creates hundreds of deployable units with no shared code.

**Fix:** Group related endpoints into a single function or use a framework (Powertools, Chalice, SST) that deploys a cohesive service.

---

## When Serverless Doesn't Fit

| Workload | Why Serverless Fails | Alternative |
|----------|---------------------|-------------|
| Long-running processes (> 15 min) | Execution time limits | Containers, Step Functions |
| Sustained high throughput | Per-invocation pricing is expensive at scale | Containers or VMs |
| Stateful workloads | No local state between invocations | Containers with persistent storage |
| Low-latency requirements (< 10ms) | Cold starts break the SLO | Containers (always warm) |
| GPU/ML inference | FaaS doesn't offer GPU | GPU instances or managed ML |
| WebSocket / long-lived connections | Stateless model doesn't support | API Gateway WebSocket or containers |
| Monolithic applications | Not designed for event-driven decomposition | Containers or PaaS |

### The Crossover Point

```
Cost ($)
│
│         ╱  Containers (fixed base + per-container)
│        ╱
│       ╱
│      ╱
│     ╱ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
│    ╱           Serverless (per-invocation)
│   ╱        ╱
│  ╱     ╱
│ ╱  ╱
│╱╱
├─────────────────────────────────── Requests/month
│        ↑
│   Crossover point
│   (~1M–10M requests/month, varies)
```

Below the crossover, serverless is cheaper (often free). Above it, containers win on per-unit economics.

---

## Key Takeaways

- Serverless is ideal for event-driven, bursty, and low-traffic workloads where per-invocation pricing and scale-to-zero matter
- Cold starts are the main latency concern — mitigate with lightweight runtimes, minimal dependencies, or provisioned concurrency
- Use workflow engines (Step Functions, Temporal) for multi-step processes — don't chain functions synchronously
- Serverless databases extend the pay-per-use model to data storage; use them for unpredictable or variable traffic
- Avoid the distributed monolith: if functions are tightly coupled, they should be a single function or a container
- Serverless has a cost crossover point — at sustained high volume, containers are more economical
