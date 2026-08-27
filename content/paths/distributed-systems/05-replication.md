---
title: Replication
weight: 5
---

# Replication

Replication — maintaining copies of data on multiple nodes — serves three goals: **fault tolerance** (surviving node failures), **lower latency** (serving reads from nearby replicas), and **higher throughput** (distributing load). The challenge is keeping replicas consistent while maximising performance and availability.

This section covers the three fundamental replication topologies, the consistency problems each introduces, and the strategies for resolving conflicts.

## Single-Leader Replication

In single-leader (also called primary-backup or master-slave) replication, one node is designated the **leader**. All writes go to the leader, which replicates changes to followers.

```
Clients (writes)
      │
      ▼
   ┌────────┐     replication     ┌────────┐
   │ Leader  │──────────────────► │Follower│
   │ (R/W)   │──────────────────► │  (R)   │
   └────────┘                     └────────┘
                                  ┌────────┐
              ──────────────────► │Follower│
                                  │  (R)   │
                                  └────────┘
```

### Synchronous vs Asynchronous Replication

| Mode | Behaviour | Tradeoff |
|------|----------|----------|
| **Synchronous** | Leader waits for follower ACK before confirming write | Durable, but one slow follower blocks all writes |
| **Asynchronous** | Leader confirms immediately, replicates in background | Fast, but follower may lag (data loss on leader crash) |
| **Semi-synchronous** | Leader waits for 1 follower ACK, rest async | Compromise: one guaranteed replica, no blocking on all |

Most production databases use **semi-synchronous** replication. PostgreSQL calls this `synchronous_commit`; MySQL uses semi-sync replication plugins.

### Replication Lag Problems

When followers serve reads asynchronously, clients may observe stale data. This creates three classic anomalies:

#### Reading Your Own Writes

```
Client A writes to Leader: x = 42
Client A reads from Follower: x = 0    ← stale! follower hasn't caught up

Fix: route reads to the leader (or a caught-up replica) after a write
     OR use session-sticky routing so the same client hits the same replica
```

#### Monotonic Reads

```
Client reads from Follower 1: x = 42   (up to date)
Client reads from Follower 2: x = 0    ← went backward! follower 2 is behind

Fix: pin each client to a single follower for the duration of their session
```

#### Consistent Prefix Reads

```
Client observes:
    Answer: "Fine, thanks!"       (replicated first)
    Question: "How are you?"      (replicated second, from a different partition)

Fix: ensure causally related writes go through the same partition
     OR use causal consistency (Section 3)
```

### Failover

When the leader fails, a follower must be promoted:

1. Detect leader failure (heartbeat timeout)
2. Choose a new leader (most up-to-date follower, or via consensus)
3. Reconfigure clients and remaining followers to use the new leader

**Risks during failover:**

| Risk | Consequence |
|------|------------|
| Async follower promoted | Writes acknowledged by old leader but not yet replicated are lost |
| Split-brain | Two nodes believe they are leader; both accept writes |
| Stale reads during transition | Clients may read from followers that are behind the new leader |

## Multi-Leader Replication

In multi-leader (also called active-active or master-master) replication, multiple nodes accept writes independently and replicate changes to each other.

```
┌────────┐         replication         ┌────────┐
│Leader A │◄──────────────────────────►│Leader B │
│ (R/W)   │         (bidirectional)     │ (R/W)   │
└────────┘                             └────────┘
    │                                      │
    ▼                                      ▼
┌────────┐                             ┌────────┐
│Follower│                             │Follower│
└────────┘                             └────────┘
```

### When to Use Multi-Leader

| Use case | Why |
|----------|-----|
| Multi-datacenter deployment | Each DC has a local leader; avoids cross-DC write latency |
| Offline-capable clients | Each device is a "leader" that syncs when reconnected (e.g., CouchDB, Notion) |
| Collaborative editing | Each editor instance accepts local changes immediately |

### The Conflict Problem

When two leaders concurrently modify the same data, a **write conflict** occurs. This is the fundamental challenge of multi-leader replication.

```
Leader A: UPDATE users SET name='Alice' WHERE id=1   (at T=100)
Leader B: UPDATE users SET name='Alicia' WHERE id=1  (at T=101)

After replication: which value wins?
```

### Conflict Resolution Strategies

| Strategy | How it works | Tradeoff |
|----------|-------------|----------|
| **Last-writer-wins (LWW)** | Highest timestamp wins | Simple, but silently discards writes. Depends on clock accuracy |
| **Per-replica priority** | Higher-priority replica's write wins | Deterministic, but lower-priority writes are always lost |
| **Merge values** | Combine both writes (e.g., union of sets) | Works for some data types, meaningless for others |
| **Record conflict** | Store both versions, let application resolve | Most flexible, but pushes complexity to the application |
| **CRDTs** | Use data structures that merge automatically | Best for supported types (Section 8) |

## Leaderless Replication (Quorum)

In leaderless replication, clients write to **multiple replicas directly** and read from multiple replicas, using quorum rules to ensure consistency.

```
Client write ──────► Node 1  ✓
              ──────► Node 2  ✓
              ──────► Node 3  ✗ (down)

Client read  ◄────── Node 1  x=42
             ◄────── Node 2  x=42
             ◄────── Node 3  (down)

Write quorum: W = 2 of 3  ✓
Read quorum:  R = 2 of 3  ✓
W + R > N → guaranteed to overlap → read sees latest write
```

### Quorum Condition

For a cluster of **N** replicas:

```
W + R > N

Where:
  W = number of nodes that must acknowledge a write
  R = number of nodes that must respond to a read
  N = total replicas
```

| Configuration | W | R | Tolerance | Use case |
|--------------|---|---|-----------|----------|
| Strict majority | ⌈(N+1)/2⌉ | ⌈(N+1)/2⌉ | Read & write survive ⌊N/2⌋ failures | Default (balanced) |
| Write-heavy | N | 1 | Reads from any node | High read throughput, low write availability |
| Read-heavy | 1 | N | Writes to any node | High write throughput, low read availability |

### Sloppy Quorums and Hinted Handoff

When a node is temporarily unavailable, a **sloppy quorum** allows writes to be sent to a different node that is not normally responsible for that data. When the original node recovers, the substitute hands off the data.

```
Normal: Write to nodes {1, 2, 3}
Node 3 is down → Sloppy quorum: Write to nodes {1, 2, 4}
Node 3 recovers → Node 4 sends the data to Node 3 (hinted handoff)
```

Sloppy quorums increase write availability but weaken consistency — the W + R > N condition no longer guarantees overlap, because the write went to a different set of nodes.

### Read Repair and Anti-Entropy

Leaderless systems need mechanisms to converge stale replicas:

| Mechanism | How it works | Latency |
|-----------|-------------|---------|
| **Read repair** | During a read, if replicas disagree, the client writes the latest value back to stale replicas | On-read (lazy) |
| **Anti-entropy** | Background process compares replicas and copies missing data | Periodic (eventually) |

Read repair is fast but only fixes data that is actually read. Anti-entropy is comprehensive but adds background load. Most systems use both.

## Chain Replication

Chain replication (van Renesse & Schneider, 2004) arranges replicas in a **chain**. Writes enter at the head and propagate to the tail. Reads are served exclusively from the tail.

```
Client write → [Head] → [Middle] → [Tail] → ACK to client
                                      ↑
                              Client read ─┘
```

### Properties

| Property | Chain replication | Quorum (Raft-style) |
|----------|------------------|-------------------|
| Write path | Sequential through chain | Leader to quorum in parallel |
| Read path | Tail only (always consistent) | Leader or quorum |
| Write latency | Higher (sequential) | Lower (parallel) |
| Read latency | Low (single node, no quorum) | Low to medium |
| Read throughput | Very high (single node, always up to date) | Requires leader or quorum |
| Failure handling | Requires chain reconfiguration | Leader election |

Chain replication is used in **HDFS** (for block replication) and **Azure Storage**. It offers simple, strong consistency for reads at the cost of higher write latency.

### CRAQ: Chain Replication with Apportioned Queries

**CRAQ** extends chain replication by allowing reads from any node in the chain, not just the tail. A node serves a read immediately if its latest version is committed (matches the tail); otherwise, it queries the tail for the committed version. This dramatically increases read throughput while preserving strong consistency.

## Conflict Resolution in Detail

When concurrent writes conflict (multi-leader or leaderless), the system must resolve them.

### Version Vectors for Conflict Detection

Version vectors (a generalisation of vector clocks, Section 2) track which writes each replica has seen:

```
Replica A: {A:3, B:2}  — has seen 3 of its own writes and 2 from B
Replica B: {A:2, B:4}  — has seen 2 from A and 4 of its own writes

These are concurrent: A has seen more of A's writes, B has seen more of B's
→ conflict detected
```

### Convergent Conflict Resolution

For conflicts that must be resolved automatically:

| Approach | Mechanism | Data loss? |
|----------|-----------|-----------|
| **LWW (Last Writer Wins)** | Compare timestamps, keep highest | Yes — lower-timestamp writes discarded |
| **Multi-value (siblings)** | Return all conflicting values to client | No — but client must resolve |
| **Application-defined merge** | Custom function (e.g., union for shopping cart) | Depends on function |
| **CRDTs** | Mathematically convergent merge | No — by construction (Section 8) |

Dynamo-style databases (Riak, Cassandra) use a combination of LWW and client-side resolution. Riak can return "siblings" (multiple conflicting values) and let the application choose.

## Key Takeaways

- Single-leader replication routes all writes through one node — simple consistency, but the leader is a bottleneck and failover is risky
- Asynchronous replication introduces read-your-writes, monotonic reads, and consistent prefix reads anomalies — session guarantees mitigate these
- Multi-leader replication enables multi-datacenter and offline-first architectures but introduces write conflicts that must be detected and resolved
- Leaderless (quorum) replication writes to and reads from multiple nodes — the quorum condition W + R > N ensures reads overlap with writes
- Sloppy quorums increase availability but weaken the quorum guarantee — hinted handoff repairs data when nodes recover
- Chain replication offers strong consistency reads from the tail with high throughput, at the cost of sequential write latency
- Conflict resolution strategies range from LWW (simple, lossy) to CRDTs (lossless, mathematically convergent) — the choice depends on the data model and tolerance for data loss
