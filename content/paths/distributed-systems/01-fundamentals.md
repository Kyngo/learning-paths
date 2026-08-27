---
title: Distributed Systems Fundamentals
weight: 1
---

# Distributed Systems Fundamentals

A distributed system is a collection of independent computers that appears to its users as a single coherent system. This deceptively simple definition hides enormous complexity — complexity that has broken production systems at every scale, from startups to hyperscalers.

This section covers **why** distributed systems are fundamentally harder than single-node systems, the theoretical limits that constrain what is possible, and the models we use to reason about them.

## Why Distribution Is Hard

A single-node program has properties we take for granted:

- **Total order of operations** — instructions execute in a defined sequence
- **Shared memory** — all components see the same state instantly
- **Fail-stop behaviour** — the process either works or crashes entirely

In a distributed system, **none of these hold**. Messages arrive out of order, nodes observe different states at the same instant, and a node can appear alive to some peers and dead to others.

### The Two Generals Problem

The Two Generals Problem is the earliest illustration of the impossibility of reliable communication over unreliable channels.

```
General A                          General B
   |                                  |
   |--- "Attack at dawn" ----------->|  (might be lost)
   |                                  |
   |<-- "Acknowledged" --------------|  (might be lost)
   |                                  |
   |--- "Ack of your ack" ---------->|  (might be lost)
   |         ...infinite regress...   |
```

No finite number of message exchanges can guarantee both generals know the other will attack. This is **not** an engineering problem with a clever solution — it is a proven impossibility when the communication channel is unreliable.

**Practical implication:** you can never be certain a remote node received your message. Every distributed protocol must deal with this uncertainty.

## Fallacies of Distributed Computing

In 1994, Peter Deutsch (with additions by James Gosling) codified eight assumptions that developers incorrectly make about networks:

| # | Fallacy | Reality |
|---|---------|---------|
| 1 | The network is reliable | Packets are dropped, duplicated, and reordered routinely |
| 2 | Latency is zero | Cross-datacenter round trips are 30–150 ms; tail latency is worse |
| 3 | Bandwidth is infinite | Saturated links cause backpressure and cascading slowdowns |
| 4 | The network is secure | Every network hop is a potential attack surface |
| 5 | Topology doesn't change | Nodes join, leave, and move between racks and regions |
| 6 | There is one administrator | Multiple teams, providers, and policies govern the network |
| 7 | Transport cost is zero | Serialisation, encryption, and cross-region transfer have real cost |
| 8 | The network is homogeneous | Different hardware, OS versions, and clock sources coexist |

Every fallacy maps to a class of production incident. Designing distributed systems well means internalising these realities, not hoping they won't apply to you.

## Network Partitions

A **network partition** occurs when a subset of nodes cannot communicate with another subset, even though both subsets are individually functional.

```
┌─────────────────┐         ╳ PARTITION ╳         ┌─────────────────┐
│  Partition A     │         ╳           ╳         │  Partition B     │
│                  │         ╳           ╳         │                  │
│  Node 1  Node 2  │◄───────╳───────────╳────────►│  Node 3  Node 4  │
│                  │         ╳           ╳         │                  │
└─────────────────┘         ╳           ╳         └─────────────────┘
```

During a partition, each side must independently decide how to behave:

- **Continue serving requests** — risking inconsistent state across partitions (availability over consistency)
- **Refuse requests** — preserving consistency at the cost of availability

This tradeoff is formalised by the CAP theorem (covered in [Databases: NoSQL]({{< relref "/paths/databases/09-nosql" >}})). Here we focus on the mechanics: partitions are not rare edge cases. Cloud providers report network partitions regularly, even within a single availability zone.

### Partial Partitions

Not all partitions are symmetric. A **partial partition** occurs when Node A can reach Node B, and Node B can reach Node C, but Node A **cannot** reach Node C directly.

```
Node A ◄──────► Node B ◄──────► Node C
   │                                │
   └──────────── ╳ ─────────────────┘
              (blocked)
```

Partial partitions are insidious because they break the assumption of transitive connectivity. They cause split-brain scenarios even in systems designed to tolerate full partitions.

## Partial Failures

In a single-node system, failures are **total** — the entire process crashes. In a distributed system, failures are **partial** — some components fail while others continue operating. This creates states that are far harder to reason about:

- A request may have been processed by the remote node, but the response was lost
- A node may be running but so slow it is indistinguishable from a dead node
- A disk write may have succeeded on 2 of 3 replicas

### The Grey Failure Problem

Grey failures — where a component is degraded but not fully failed — are the most common source of distributed system outages. Examples:

| Failure type | Symptom | Detection difficulty |
|-------------|---------|---------------------|
| Crash failure | Node stops responding | Easy (timeout) |
| Omission failure | Node drops some messages | Medium (intermittent) |
| Timing failure | Node responds too slowly | Hard (latency vs failure?) |
| Byzantine failure | Node behaves arbitrarily | Very hard (lies are hard to detect) |

Most practical systems assume **crash-recovery** failures (nodes either work correctly or crash and eventually restart) and do not handle Byzantine faults. Byzantine fault tolerance (BFT) is primarily relevant to blockchain and safety-critical systems.

## FLP Impossibility Result

The **Fischer-Lynch-Paterson (FLP) impossibility result** (1985) is one of the most important theoretical results in distributed computing.

> **FLP Theorem:** In an asynchronous system where even a single process can crash, no deterministic consensus algorithm can guarantee termination in all executions.

### What This Means

- **Asynchronous system** — no bound on message delivery time or processing speed
- **Single crash** — just one node failing is enough to break consensus
- **Deterministic** — the algorithm cannot use randomness
- **Guarantee termination** — it is impossible to ensure the algorithm always finishes

### What This Does NOT Mean

FLP does **not** mean consensus is impossible in practice. It means you cannot have all three of:

1. **Safety** — the algorithm never produces a wrong result
2. **Liveness** — the algorithm always terminates
3. **Fault tolerance** — the algorithm tolerates at least one crash

Practical consensus algorithms (Paxos, Raft) sacrifice guaranteed liveness — they may stall under certain failure patterns, but they never produce incorrect results. In practice, liveness violations are temporary and resolve when the network stabilises.

## System Models

To reason about distributed algorithms, we define **system models** — assumptions about how the system behaves. Every algorithm operates within a model, and using an algorithm outside its assumed model voids its guarantees.

### Timing Models

| Model | Message delivery | Processing speed | Practical example |
|-------|-----------------|-----------------|-------------------|
| **Synchronous** | Bounded (known upper limit) | Bounded | Single-rack cluster with dedicated network |
| **Asynchronous** | Unbounded (no time guarantees) | Unbounded | The internet; any network with arbitrary delays |
| **Partially synchronous** | Eventually bounded (after some unknown time) | Eventually bounded | Most real datacenter networks |

Most practical algorithms assume **partial synchrony** — the system may behave asynchronously for a while, but eventually messages are delivered within some bound. This is a realistic model for datacenter networks.

### Failure Models

| Model | Assumption | Used by |
|-------|-----------|---------|
| **Crash-stop** | Failed nodes never recover | Theoretical proofs |
| **Crash-recovery** | Failed nodes may restart with durable state | Paxos, Raft, most production systems |
| **Byzantine** | Failed nodes may behave arbitrarily (including maliciously) | BFT protocols, blockchains |
| **Omission** | Nodes may fail to send or receive some messages | Some network models |

### Combining Models

A complete system model specifies both a timing model and a failure model:

```
System Model = Timing Model + Failure Model

Example: "Partially synchronous, crash-recovery"
 → Messages are eventually delivered within a bound
 → Nodes may crash and restart with persistent state
 → This is the model assumed by Raft and most production databases
```

## The Role of Determinism

An important distinction in distributed algorithms is between **deterministic** and **randomised** approaches:

- **Deterministic algorithms** make the same decisions given the same inputs — FLP applies to these
- **Randomised algorithms** use coin flips to break symmetry — they can circumvent FLP by providing probabilistic liveness guarantees (e.g., randomised consensus)

In practice, leader election often uses randomised timeouts (as in Raft) to avoid the liveness trap identified by FLP.

## The Spectrum of Distribution

Not all distributed systems face the same challenges. The difficulty scales with the deployment model:

| Deployment | Example | Key challenge |
|-----------|---------|---------------|
| Single-machine, multi-process | Microservices on one host | Shared clock, fast IPC, correlated failures |
| Single-datacenter | Cluster in one AZ | Low latency (<1 ms), rare partitions, single admin |
| Multi-datacenter | Active-active across regions | High latency (30–150 ms), partitions, regulatory constraints |
| Edge/global | CDN, mobile, IoT | Extreme latency, frequent disconnection, heterogeneous hardware |

The further right you go, the more the fallacies of distributed computing bite, and the more you need the theory covered in this path.

## Key Takeaways

- Distributed systems lack shared memory, global clock, and total ordering — properties that single-node systems provide for free
- The eight fallacies of distributed computing describe assumptions that cause production failures when violated
- Network partitions are normal, not exceptional — systems must be designed to handle them
- Partial failures (grey failures) are harder to detect and handle than total failures
- The FLP impossibility result proves that deterministic consensus cannot guarantee termination in asynchronous systems with even one crash — but practical algorithms work around this by assuming partial synchrony
- System models (timing + failure) define the assumptions under which an algorithm is correct — using an algorithm outside its model voids its guarantees
- Most production systems assume a partially synchronous, crash-recovery model
