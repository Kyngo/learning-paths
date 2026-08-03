---
title: "Architectural Patterns"
weight: 7
---

# Architectural Patterns

Architectural patterns are proven structural templates for organizing software systems. Each pattern makes different tradeoffs between complexity, scalability, flexibility, and maintainability. Choosing the right one depends on your constraints and quality attributes.

---

## Layered Architecture

The most common pattern — organizes code into horizontal layers with strict dependency rules:

```
┌─────────────────────────────┐
│   Presentation Layer        │  UI, API endpoints
├─────────────────────────────┤
│   Business Logic Layer      │  Domain rules, workflows
├─────────────────────────────┤
│   Persistence Layer         │  Data access, repositories
├─────────────────────────────┤
│   Database Layer            │  Storage engines
└─────────────────────────────┘
     Dependencies flow DOWN only
```

| Aspect | Details |
|--------|---------|
| **Principle** | Each layer only depends on the layer directly below |
| **Variants** | Strict (only adjacent layers) vs relaxed (skip layers) |
| **Strengths** | Simple, well-understood, good separation of concerns |
| **Weaknesses** | Monolithic deployments, changes ripple through layers |
| **Best for** | Traditional business applications, CRUD systems, small teams |

### Sinkhole Anti-Pattern

When requests pass through layers without adding value — just forwarding calls. If > 20% of requests are sinkholes, the architecture is over-layered.

---

## Pipe-and-Filter

Data flows through a pipeline of independent processing stages:

```
Input → [Filter A] → [Filter B] → [Filter C] → Output
              │             │             │
         Transform      Validate       Enrich
```

| Aspect | Details |
|--------|---------|
| **Principle** | Each filter transforms data independently; pipes connect them |
| **Strengths** | Highly composable, easy to add/remove/reorder stages |
| **Weaknesses** | Not suited for interactive systems; overhead in serialization |
| **Best for** | Data processing, ETL, compilers, Unix command pipelines |

### Example: Data Pipeline

```
Raw CSV → [Parse] → [Validate] → [Transform] → [Enrich] → [Load to DB]
                         │
                    [Error Queue]
```

### Variants

| Variant | Description |
|---------|-------------|
| Linear pipeline | Strict sequential processing |
| Fork/join | Parallel branches that merge |
| Feedback loop | Output routed back as input |

---

## Broker Pattern

A central broker mediates communication between decoupled components:

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Producer │────→│  Broker  │────→│ Consumer │
└──────────┘     │(mediator)│     └──────────┘
┌──────────┐     │          │     ┌──────────┐
│ Producer │────→│          │────→│ Consumer │
└──────────┘     └──────────┘     └──────────┘
```

| Aspect | Details |
|--------|---------|
| **Principle** | Producers and consumers don't know about each other |
| **Strengths** | Loose coupling, dynamic routing, protocol translation |
| **Weaknesses** | Broker is SPOF, added latency, complexity |
| **Best for** | Message-oriented middleware, service buses, event systems |

### Implementations

| Technology | Type | Use Case |
|------------|------|----------|
| RabbitMQ | Message broker | Task queues, RPC |
| Apache Kafka | Event streaming | Event sourcing, CDC |
| AWS SNS/SQS | Cloud messaging | Serverless event-driven |
| Redis Pub/Sub | In-memory broker | Real-time notifications |

---

## Peer-to-Peer

All nodes are equal — each can be both client and server:

```
    ┌───┐     ┌───┐
    │ A │─────│ B │
    └─┬─┘     └─┬─┘
      │    ╲    │
      │     ╲   │
    ┌─┴─┐   ┌┴──┐
    │ C │───│ D │
    └───┘   └───┘
```

| Aspect | Details |
|--------|---------|
| **Principle** | No central authority; nodes discover and communicate directly |
| **Strengths** | Fault-tolerant, scalable, no SPOF |
| **Weaknesses** | Complex coordination, eventual consistency, discovery overhead |
| **Best for** | File sharing (BitTorrent), blockchain, distributed databases |

### P2P Variants

| Variant | Structure | Example |
|---------|-----------|---------|
| Pure P2P | No central node at all | Early Gnutella |
| Hybrid P2P | Some supernodes for coordination | Skype (legacy), BitTorrent trackers |
| Structured P2P | Deterministic topology (DHT) | Chord, Kademlia, IPFS |

---

## Space-Based Architecture

Designed for extreme scalability by eliminating the database as a bottleneck:

```
┌────────────────────────────────────────────────┐
│              Virtualized Middleware             │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐     │
│  │ PU 1 │  │ PU 2 │  │ PU 3 │  │ PU N │     │
│  │      │  │      │  │      │  │      │     │
│  │ App  │  │ App  │  │ App  │  │ App  │     │
│  │ Cache │  │ Cache │  │ Cache │  │ Cache │     │
│  └──────┘  └──────┘  └──────┘  └──────┘     │
│         ↕ Data Replication Grid ↕              │
├────────────────────────────────────────────────┤
│  Messaging Grid │ Data Grid │ Processing Grid  │
├────────────────────────────────────────────────┤
│         Async Database Write-Behind            │
│                    ┌────┐                      │
│                    │ DB │                      │
│                    └────┘                      │
└────────────────────────────────────────────────┘
```

| Aspect | Details |
|--------|---------|
| **Principle** | Processing units hold data in-memory; no shared DB for hot path |
| **Strengths** | Near-infinite scalability, low latency, handles spikes |
| **Weaknesses** | Complex, eventual consistency, costly, hard to debug |
| **Best for** | Concert ticketing, flash sales, real-time bidding, gaming |

### Key Components

| Component | Purpose |
|-----------|---------|
| Processing Unit (PU) | Self-contained app + in-memory data grid |
| Data replication | Sync state across PUs |
| Messaging grid | Route requests to PUs |
| Data pump | Async write-behind to persistent store |

---

## Serverless Architecture

Functions execute in response to events — no server management:

```
┌──────────┐     ┌───────────────┐     ┌──────────┐
│  Event   │────→│   Function    │────→│  Output  │
│ (trigger)│     │ (ephemeral)   │     │ (storage)│
└──────────┘     └───────────────┘     └──────────┘

Events: HTTP request, queue message, file upload, schedule
```

| Aspect | Details |
|--------|---------|
| **Principle** | Write functions; cloud handles scaling, availability, infrastructure |
| **Strengths** | Zero ops, pay-per-use, auto-scaling, fast to deploy |
| **Weaknesses** | Cold starts, vendor lock-in, debugging difficulty, state management |
| **Best for** | Event-driven workloads, APIs, scheduled tasks, glue logic |

### Serverless Patterns

| Pattern | Description |
|---------|-------------|
| API backend | API Gateway + Lambda + DynamoDB |
| Event processor | S3/SQS trigger + Lambda + downstream |
| Scheduled job | EventBridge schedule + Lambda |
| Choreography | Events flow between functions via queues/topics |

### Limitations

| Concern | Challenge |
|---------|-----------|
| Cold start | 100ms-several seconds on first invocation |
| Duration | Max 15 minutes (AWS Lambda) |
| State | Must externalize to DB/cache/S3 |
| Testing | Hard to replicate cloud environment locally |
| Vendor lock-in | Tight coupling to provider's event model |

---

## Pattern Comparison

| Pattern | Scalability | Simplicity | Deployability | Fault Tolerance | Cost |
|---------|-------------|------------|---------------|-----------------|------|
| Layered | Low | High | Low (monolith) | Low | Low |
| Pipe-and-Filter | Medium | Medium | Medium | Medium | Low |
| Broker | High | Medium | High | Medium (SPOF) | Medium |
| Peer-to-Peer | Very High | Low | High | Very High | Low |
| Space-Based | Very High | Low | High | High | High |
| Serverless | Very High | Medium | Very High | High | Variable |

### Decision Matrix

| If you need... | Consider |
|----------------|----------|
| Simple CRUD application | Layered |
| Data transformation pipeline | Pipe-and-Filter |
| Decoupled async communication | Broker |
| Extreme fault tolerance, no central point | Peer-to-Peer |
| Extreme scalability for burst traffic | Space-Based |
| Minimal ops, event-driven workloads | Serverless |
| Independent team deployment | Microservices (see previous chapter) |

---

## Combining Patterns

Real systems combine patterns. A modern e-commerce platform might use:

- **Layered** within each microservice
- **Broker** (Kafka) between services
- **Serverless** for event-driven background tasks
- **Pipe-and-filter** for data ingestion
- **Space-based** for the checkout during flash sales

The art is knowing which pattern solves which problem at which level of the system.

---

## Key Takeaways

1. **No pattern is universally best** — each optimizes for different quality attributes.
2. **Layered architecture** is the safe default for simple applications but resists scaling.
3. **Pipe-and-filter** excels for data processing; think Unix philosophy.
4. **Broker** decouples producers from consumers but introduces a SPOF.
5. **Space-based** solves extreme scale but at extreme complexity and cost.
6. **Serverless** minimizes operations but introduces cold starts and vendor lock-in.
7. Real systems **combine patterns** at different levels — choose per component, not per system.
