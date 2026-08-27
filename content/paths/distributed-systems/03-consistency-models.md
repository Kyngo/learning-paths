---
title: Consistency Models
weight: 3
---

# Consistency Models

When data is replicated across multiple nodes, different replicas can disagree about the current state. A **consistency model** defines the rules about what values a read operation can return and when writes become visible. Choosing the right consistency model is one of the most consequential design decisions in a distributed system — it determines the tradeoff between correctness, performance, and availability.

This section covers the hierarchy of consistency models from strongest to weakest, the guarantees each provides, and when to use them.

> **Note:** The CAP theorem, which constrains the relationship between consistency, availability, and partition tolerance, is covered in [Databases: NoSQL]({{< relref "/paths/databases/09-nosql" >}}). This section focuses on what each consistency level actually means.

## The Consistency Hierarchy

```
Strongest ┌──────────────────────────┐
          │     Linearizability       │  "Behaves like a single copy"
          ├──────────────────────────┤
          │  Sequential Consistency   │  "Same global order, maybe not real-time"
          ├──────────────────────────┤
          │   Causal Consistency      │  "Respects cause and effect"
          ├──────────────────────────┤
          │  Session Guarantees       │  "Sane behaviour within a session"
          ├──────────────────────────┤
          │  Eventual Consistency     │  "Converges... eventually"
          ├──────────────────────────┤
          │ Strong Eventual Consistency│ "Converges, deterministically"
Weakest   └──────────────────────────┘
```

Stronger models are easier to reason about but harder (slower, less available) to implement. Weaker models offer better performance and availability but push complexity to the application.

## Linearizability

Linearizability (also called **strong consistency** or **atomic consistency**) is the strongest single-object consistency model.

### Definition

A system is linearizable if every operation appears to take effect **atomically** at some point between its invocation and its response, and this point is consistent with real-time ordering.

### What This Means

```
Client A:  |--- write(x=1) ---|
Client B:                   |--- read(x) ---|  must return 1
Client C:           |--- read(x) ---|          may return 0 or 1
                                                (write might not have taken effect yet)

Timeline:  ─────────────────────────────────────────►
                              ↑
                     linearization point
                     (write "takes effect" here)
```

Once any client observes a write, **all subsequent reads from any client** must see that write or a later one. There is no going back.

### Properties

| Property | Guaranteed? |
|----------|------------|
| Real-time ordering respected | ✅ |
| Reads always return latest write | ✅ |
| All clients see the same order | ✅ |
| Available during network partitions | ❌ (requires majority quorum) |

### Cost of Linearizability

- Requires coordination on **every read and write** (typically a quorum or leader)
- Cross-datacenter linearizability adds one RTT per operation (50–150 ms)
- Cannot be provided during a network partition (this is the "C" in CAP)

### Where It's Used

- **etcd, ZooKeeper, Consul** — configuration and coordination stores
- **Spanner** — Google's globally distributed database (using TrueTime)
- **Single-leader databases** — reads from the leader are linearizable

## Sequential Consistency

Sequential consistency is slightly weaker than linearizability — it does not require real-time ordering.

### Definition

A system is sequentially consistent if the result of any execution is the same as if all operations were executed in **some sequential order**, and the operations of each individual process appear in the order specified by their program.

### Difference from Linearizability

```
Real time:   ──────────────────────────────────────►

Client A:    write(x=1)         read(x)→?
Client B:              write(x=2)        read(x)→?

Linearizable: If write(x=2) completed before read(x), read must return 2.

Sequentially consistent: Operations can be reordered as long as each client's
order is preserved. A valid sequential order might be:
    write(x=1), read(x)→1, write(x=2), read(x)→2
Even if write(x=2) finished first in real time.
```

| Property | Linearizable | Sequentially consistent |
|----------|-------------|----------------------|
| All clients see same order | ✅ | ✅ |
| Order respects real-time | ✅ | ❌ |
| Per-process order preserved | ✅ | ✅ |

### Where It's Used

Sequential consistency is the model provided by **ZooKeeper reads** (reads may be served by followers and lag behind the leader, but each client sees a consistent progression).

## Causal Consistency

Causal consistency respects the **happens-before** relation (Section 2) but allows concurrent operations to be observed in different orders by different clients.

### Definition

A system is causally consistent if all processes see causally related operations in the same order. Concurrent operations may be seen in different orders by different processes.

### Example

```
Client A:  write(x=1)
Client B:  read(x)→1, then write(y=2)    (y=2 is causally dependent on x=1)

Causally consistent:
  All clients must see write(x=1) before write(y=2)
  because reading x=1 caused the write of y=2.

Client C:  write(z=3)                     (concurrent with all of the above)

  Different clients may see write(z=3) at different points in the order,
  as long as x=1 always appears before y=2.
```

### Properties

| Property | Guaranteed? |
|----------|------------|
| Causally related ops in same order everywhere | ✅ |
| Concurrent ops in same order everywhere | ❌ |
| Available during partitions | ✅ (this is the strongest model that allows full availability) |

### Why Causal Consistency Matters

Causal consistency is the **strongest consistency model achievable without sacrificing availability** during network partitions. This makes it particularly valuable for geo-distributed systems.

Implementations typically use vector clocks or hybrid logical clocks to track causal dependencies.

### Where It's Used

- **MongoDB** (with causal consistency sessions enabled)
- **COPS** (research system from Lloyd et al., 2011)
- **Riak** (via dotted version vectors)

## Session Guarantees

Session guarantees provide consistency within a single client's session, without global ordering guarantees. They are often combined with eventual consistency to prevent the most confusing behaviours.

| Guarantee | Definition | Without it... |
|-----------|-----------|---------------|
| **Read Your Writes** | A read after a write by the same client always sees that write | You save a form, refresh the page, and see stale data |
| **Monotonic Reads** | Successive reads by the same client never go backward | You see a comment, refresh, and the comment disappears, then reappears |
| **Monotonic Writes** | Writes by the same client are applied in order | A password change followed by a login attempt uses the old password |
| **Writes Follow Reads** | A write that follows a read reflects the state seen by that read | You reply to a message that hasn't been delivered to your replica yet |

### Implementation

Session guarantees are typically implemented by **session tokens** that track the client's most recent read/write position. The client presents the token with each request, and the serving replica ensures it is at least as up-to-date as the token indicates (or redirects to a replica that is).

```
Client → Request + SessionToken(version=42)
         ↓
Replica: "My version is 40. I need to catch up or redirect."
         ↓
         Option A: Wait for replication to reach version 42
         Option B: Forward request to a replica at version ≥ 42
```

## Eventual Consistency

Eventual consistency is the weakest useful consistency model.

### Definition

If no new writes are made, all replicas will **eventually** converge to the same value. The system provides no bound on how long convergence takes.

### What It Does NOT Guarantee

- When replicas will converge
- What intermediate values a client might read
- That two clients reading "at the same time" see the same value
- That a client's own writes are immediately visible

### When It's Acceptable

Eventual consistency works well when:

- **Stale reads are tolerable** (social media feeds, analytics dashboards)
- **High availability is critical** (DNS, CDN edge caches)
- **Writes are commutative** (counters, sets — leading to CRDTs, Section 8)

### The Problem with "Eventual"

The word "eventual" is often a euphemism for "we don't know when." In practice, convergence time depends on replication lag, which depends on network conditions, load, and the anti-entropy mechanism in use. Systems that advertise "eventual consistency" must be tested under realistic conditions to understand actual convergence times.

## Strong Eventual Consistency (SEC)

Strong eventual consistency strengthens eventual consistency with a determinism guarantee.

### Definition

A system provides SEC if:

1. **Eventual delivery:** every update delivered to one correct replica is eventually delivered to all correct replicas
2. **Convergence:** any two replicas that have received the same set of updates are in the same state — **immediately**, without further communication

### Difference from Eventual Consistency

| Property | Eventual | Strong Eventual |
|----------|----------|----------------|
| Replicas eventually converge | ✅ | ✅ |
| Same updates → same state (no conflicts) | ❌ | ✅ |
| Requires conflict resolution | Yes (manual or LWW) | No (by construction) |

SEC is achieved through **CRDTs** (Conflict-free Replicated Data Types, covered in Section 8) or other mathematically convergent data structures.

## Choosing a Consistency Model

| Model | Latency | Availability | Use case |
|-------|---------|-------------|----------|
| Linearizable | Highest (quorum/leader) | Lowest (unavailable during partitions) | Locks, leader election, financial transactions |
| Sequential | High | Low | Coordination services (ZooKeeper) |
| Causal | Medium | High | Social features, collaborative editing |
| Session guarantees | Low-medium | High | User-facing CRUD applications |
| Eventual | Lowest | Highest | Caches, analytics, DNS |
| Strong eventual | Low | Highest | Collaborative editing (CRDTs), counters |

### Per-Operation Consistency

Modern systems often use **different consistency levels for different operations**:

- Bank balance check: linearizable
- Transaction history view: sequential or causal
- "Likes" counter: eventual or strong eventual

This is not a weakness — it is an intentional design that applies the right tradeoff to each operation.

## Consistency vs Isolation

A common source of confusion: **consistency models** (this section) are about **replication** — what values replicas return. **Isolation levels** (serializable, snapshot isolation, read committed) are about **transactions** — how concurrent transactions interact.

| Concept | Domain | Question answered |
|---------|--------|------------------|
| Consistency model | Replication | "What can a read return across replicas?" |
| Isolation level | Transactions | "How do concurrent transactions interact?" |

Spanner combines both: it provides **external consistency** (linearizable + serializable) — the strongest possible combination.

## Key Takeaways

- Consistency models define what values reads can return when data is replicated — stronger models are easier to reason about but slower and less available
- Linearizability is the strongest model: every operation appears to happen atomically at a point in real time, and all clients agree on the order
- Sequential consistency relaxes real-time ordering but preserves per-process order — used by ZooKeeper reads
- Causal consistency is the strongest model that remains available during partitions — it respects cause-and-effect ordering
- Session guarantees (read-your-writes, monotonic reads) prevent the most confusing behaviours in eventually consistent systems
- Strong eventual consistency (SEC) guarantees that replicas with the same updates are in the same state — no conflict resolution needed — achieved through CRDTs
- Most production systems use different consistency levels for different operations, matching the tradeoff to the requirement
