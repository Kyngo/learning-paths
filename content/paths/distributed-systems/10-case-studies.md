---
title: Case Studies
weight: 10
---

# Case Studies

Theory without practice is incomplete. This section examines how six influential distributed systems apply the concepts from this path. Each case study highlights the key architecture decisions, the tradeoffs accepted, and the lessons learned.

## Amazon Dynamo

**Paper:** "Dynamo: Amazon's Highly Available Key-value Store" (DeCandia et al., 2007)

Dynamo was built for Amazon's shopping cart — a system where availability must never be sacrificed, even at the cost of consistency. It pioneered many ideas that later appeared in Cassandra, Riak, and DynamoDB.

### Architecture Decisions

| Concept | Dynamo's choice | Why |
|---------|----------------|-----|
| Partitioning | Consistent hashing with virtual nodes (Section 6) | Even distribution, minimal rebalancing |
| Replication | Leaderless, preference list of N nodes (Section 5) | No single point of failure |
| Consistency | Eventual, with sloppy quorums (Section 3, 5) | Availability over consistency |
| Conflict resolution | Vector clocks + client-side reconciliation (Section 2, 5) | Application knows best how to merge |
| Failure detection | Gossip protocol (Section 9) | Decentralised, scalable |
| Recovery | Hinted handoff + Merkle tree anti-entropy (Section 8, 9) | Fast repair + comprehensive sync |

### How Writes Work

```
Client → any node (coordinator)
    ↓
Coordinator determines preference list: {N1, N2, N3}
    ↓
Write to W nodes (e.g., W=2 of N=3)
    ↓
If a target node is down: write to next node on ring (sloppy quorum)
    Store a hint for the unavailable node (hinted handoff)
    ↓
Return success to client after W acknowledgments
```

### Conflict Resolution: Shopping Cart Example

```
User adds item A on Device 1 → version [D1:1]
User adds item B on Device 2 → version [D2:1]

Both writes succeed (different coordinators).
Versions are concurrent: [D1:1] ‖ [D2:1]

Next read returns BOTH versions (siblings).
Application merges: cart = {A, B} (union of both versions)
Writes merged cart back with new version vector.
```

### Lessons

- **Always-writable:** Dynamo never rejects a write — this is the core design constraint that drives every other decision
- **Push conflict resolution to the client:** the application (shopping cart) knows how to merge (union of items); the database cannot make that decision
- **Sloppy quorums sacrifice consistency for availability:** reads may return stale data, but the system is always available

## Google Spanner

**Paper:** "Spanner: Google's Globally-Distributed Database" (Corbett et al., 2012)

Spanner is the opposite of Dynamo — it prioritises **consistency over availability** and achieves the strongest guarantee possible: external consistency (linearizable + serializable) across the globe.

### Architecture Decisions

| Concept | Spanner's choice | Why |
|---------|-----------------|-----|
| Partitioning | Range-based (splits), auto-splitting (Section 6) | Supports SQL range scans |
| Replication | Paxos per shard (Section 4) | Strong consistency within and across datacenters |
| Consistency | External consistency via TrueTime (Section 3, 7) | Strongest possible guarantee |
| Transactions | 2PC across shards, Paxos within shards (Section 7) | ACID across globally distributed data |
| Time | TrueTime: GPS + atomic clocks, bounded uncertainty (Section 7) | Enables commit-wait protocol |
| Failure model | Crash-recovery, Paxos-based (Section 1) | Tolerates minority failures per shard |

### Read-Write Transaction Flow

```
1. Client starts transaction at coordinator (leader of first shard)
2. Reads acquire read locks on relevant shard leaders
3. Writes are buffered at the client

4. Commit:
   a. If single-shard: Paxos commit (fast path)
   b. If multi-shard: 2PC
      - Coordinator sends Prepare to participant shard leaders
      - Each participant runs Paxos to replicate the Prepare decision
      - All vote YES → Coordinator assigns commit timestamp
      - Commit-wait: wait until TrueTime.now().earliest > commit_ts
      - Send Commit to all participants

5. After commit-wait: transaction is visible to all, globally
```

### Read-Only Transactions (Lock-Free)

Spanner can serve globally consistent reads **without locks** by using TrueTime:

```
1. Client picks a read timestamp t_read = TrueTime.now().latest
2. Each shard serves data as of t_read from its Paxos log
3. No locks needed — snapshot reads from the replicated log
4. Consistent across all shards: same timestamp, same snapshot
```

### Lessons

- **Specialised hardware enables software simplicity:** TrueTime (GPS + atomic clocks) reduces clock uncertainty to ~7 ms, making commit-wait practical
- **External consistency is achievable at global scale** — but requires significant infrastructure investment
- **Paxos per shard + 2PC across shards:** a layered approach where consensus handles replication and 2PC handles atomicity

## Apache Kafka

**Origin:** Built at LinkedIn, open-sourced 2011. Now maintained by the Apache Foundation and Confluent.

Kafka is a distributed commit log — it stores ordered, durable streams of records. Its design prioritises **throughput, durability, and ordering** over low latency.

### Architecture Decisions

| Concept | Kafka's choice | Why |
|---------|---------------|-----|
| Partitioning | Hash partitioning by key, within topics (Section 6) | Parallelism + per-key ordering |
| Replication | Single-leader per partition (ISR) (Section 5) | Strong ordering within partitions |
| Consistency | Leader serves reads and writes (Section 3) | Linearizable per partition |
| Ordering | Total order within partition, no cross-partition order | Practical compromise for throughput |
| Consensus | KRaft (Raft-based) for metadata (Section 4) | Replaced ZooKeeper dependency |
| Failure handling | ISR (in-sync replicas) (Section 5) | Flexible durability guarantees |

### In-Sync Replicas (ISR)

```
Partition 0: Leader = Broker 1, Replicas = {Broker 1, Broker 2, Broker 3}

ISR = {Broker 1, Broker 2, Broker 3}  ← all caught up

Broker 3 falls behind:
ISR = {Broker 1, Broker 2}            ← Broker 3 removed from ISR

Producer writes with acks=all:
    Wait for ALL ISR members to acknowledge
    → Broker 1 and Broker 2 acknowledge → committed

Broker 3 catches up:
ISR = {Broker 1, Broker 2, Broker 3}  ← Broker 3 rejoined
```

### KRaft: Kafka's Raft Implementation

Kafka originally depended on ZooKeeper for metadata management (controller election, partition assignments, topic configuration). KRaft replaces ZooKeeper with a built-in Raft-based metadata quorum.

```
Before KRaft:
    Kafka brokers ←──── coordination ────► ZooKeeper ensemble
    (separate cluster, additional operational burden)

After KRaft:
    Kafka brokers include controller nodes
    Controller quorum uses Raft for metadata consensus
    No external dependency
```

### Lessons

- **The log is the fundamental abstraction:** Kafka's power comes from treating data as an immutable, append-only, ordered log
- **Per-partition ordering is a practical compromise:** total ordering across all data is too expensive; per-key ordering within a partition is sufficient for most use cases
- **ISR provides flexible durability:** `acks=all` ensures committed records survive leader failure (if ISR has >1 member); `acks=1` trades durability for latency

## CockroachDB

**Origin:** Built by ex-Google engineers, inspired by Spanner. Open-source, designed for global deployment without specialised hardware.

### Architecture Decisions

| Concept | CockroachDB's choice | Why |
|---------|---------------------|-----|
| Partitioning | Range-based (auto-splitting) (Section 6) | SQL compatibility, range scans |
| Replication | Raft per range (Section 4) | Strong consistency without specialised hardware |
| Consistency | Serializable isolation, linearizable reads (Section 3) | Spanner-like guarantees |
| Time | Hybrid Logical Clocks (Section 2) | Approximates TrueTime without GPS/atomic clocks |
| Transactions | Parallel commits (optimised 2PC) (Section 7) | Lower latency than standard 2PC |
| Conflict resolution | MVCC + timestamp ordering | Lock-free reads at snapshot timestamps |

### How CockroachDB Approximates Spanner

| Spanner (Google) | CockroachDB |
|-----------------|-------------|
| TrueTime (GPS + atomic) → ε ≈ 7 ms | HLC + clock offset monitoring → assumed bound ~500 ms |
| Commit-wait: 2ε ≈ 14 ms | Uncertainty interval reads: retry if uncertain |
| Guaranteed external consistency | Best-effort external consistency (stalls if clocks diverge) |

```
CockroachDB read at timestamp T:
    If data at T has write within the uncertainty interval:
        → Push read timestamp forward OR restart transaction
    If clocks are well-synchronised (NTP ≤ 250ms offset):
        → Restarts are rare in practice
```

### Lessons

- **Spanner's design is replicable without specialised hardware** — HLC + NTP provides a practical approximation
- **Raft per range:** every shard runs its own Raft group, giving strong consistency with automatic failover
- **Parallel commits:** reducing 2PC from 2 consensus rounds to 1 in the common case makes distributed transactions practical for OLTP workloads

## Apache Cassandra

**Origin:** Built at Facebook for inbox search, open-sourced 2008. Combines ideas from Dynamo (partitioning, replication) and Bigtable (data model).

### Architecture Decisions

| Concept | Cassandra's choice | Why |
|---------|-------------------|-----|
| Partitioning | Consistent hashing with vnodes (Section 6) | Dynamo-inspired, automatic rebalancing |
| Replication | Leaderless, tunable quorum (Section 5) | Flexible consistency/availability tradeoff |
| Consistency | Tunable per-query (ONE, QUORUM, ALL) (Section 3) | Application chooses per operation |
| Failure detection | Phi accrual detector (Section 9) | Adaptive to network conditions |
| Recovery | Read repair + anti-entropy repair (Merkle trees) (Section 8, 9) | Multi-layered convergence |
| Conflict resolution | LWW (last-write-wins by timestamp) (Section 5) | Simple, but can lose data |

### Tunable Consistency

```
CREATE KEYSPACE my_app WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3,
    'dc2': 3
};

-- Per-query consistency:
SELECT * FROM users WHERE id = 42;  -- consistency: ONE (fast, possibly stale)
SELECT * FROM users WHERE id = 42;  -- consistency: QUORUM (majority, consistent)
SELECT * FROM users WHERE id = 42;  -- consistency: LOCAL_QUORUM (majority in local DC)
```

| Level | Reads from | Writes to | Guarantee |
|-------|-----------|----------|-----------|
| ONE | 1 replica | 1 replica | Fastest, weakest (may be stale) |
| QUORUM | ⌈(N+1)/2⌉ replicas | ⌈(N+1)/2⌉ replicas | R + W > N → strong consistency |
| LOCAL_QUORUM | Majority in local DC | Majority in local DC | Strong within DC, eventual across DCs |
| ALL | All replicas | All replicas | Strongest, lowest availability |
| EACH_QUORUM | Quorum in each DC | Quorum in each DC | Strong in all DCs |

### Lightweight Transactions (LWT)

For operations requiring linearizability (e.g., conditional inserts), Cassandra uses Paxos:

```sql
INSERT INTO users (id, name) VALUES (42, 'Alice')
IF NOT EXISTS;  -- uses Paxos consensus (4 round trips)
```

LWT is significantly slower than normal writes (4 Paxos round trips vs 1 write). It should be used sparingly.

### Lessons

- **Tunable consistency is powerful:** letting the application choose per-query avoids one-size-fits-all tradeoffs
- **LWW is the default and it loses data:** applications must be aware that concurrent writes to the same cell result in silent data loss for the "losing" write
- **Phi accrual failure detection adapts to the environment** — Cassandra's detector performs well across diverse network conditions

## etcd and Raft

**Origin:** Built by CoreOS for Kubernetes, etcd is a distributed key-value store built on Raft. It is the coordination backbone of Kubernetes.

### Architecture Decisions

| Concept | etcd's choice | Why |
|---------|-------------|-----|
| Consensus | Raft (Section 4) | Understandable, well-tested |
| Consistency | Linearizable reads and writes (Section 3) | Coordination requires strongest guarantees |
| Replication | Raft log replication (Section 5) | Consistent replication by construction |
| Cluster size | 3 or 5 nodes (Section 4) | Small quorum, fast consensus |
| Data model | Key-value with MVCC and watch | Suitable for configuration and coordination |
| Failure tolerance | 1 failure (3 nodes) or 2 failures (5 nodes) | Sufficient for control plane |

### Linearizable Reads

etcd offers two read modes:

```
1. Serializable read (default):
   → Read from any node (may be stale if follower is behind)
   → Fast but not linearizable

2. Linearizable read:
   → Leader confirms it is still the leader (ReadIndex protocol)
   → Ensures the read reflects all committed writes
   → Additional round trip to verify leadership
```

### Watch Mechanism

etcd's watch API allows clients to subscribe to changes:

```
Watch key "/services/web":
    → Event: PUT /services/web/instance-1 = "10.0.1.5:8080"
    → Event: PUT /services/web/instance-2 = "10.0.1.6:8080"
    → Event: DELETE /services/web/instance-1

Implementation: MVCC store keeps revision history
    Watches are served from the revision log
    Clients can resume from a specific revision after disconnect
```

### etcd in Kubernetes

```
                    ┌──────────────┐
                    │  API Server   │
                    └──────┬───────┘
                           │ all cluster state
                           ▼
                    ┌──────────────┐
                    │    etcd       │
                    │  (Raft)      │
                    └──────────────┘

Kubernetes stores ALL cluster state in etcd:
    - Pod definitions and status
    - Service endpoints
    - ConfigMaps and Secrets
    - Node registrations
    - Lease objects (node heartbeats)

etcd provides:
    - Linearizable reads/writes for consistency
    - Watch for change notification (controllers)
    - Lease-based TTL for node health
```

### Lessons

- **Small cluster, strong guarantees:** etcd does not try to scale to thousands of nodes — it runs 3–5 nodes and provides linearizability, which is exactly what a coordination service needs
- **Raft makes consensus practical:** etcd's implementation of Raft is one of the most battle-tested in production, powering every Kubernetes cluster
- **Watch + MVCC enables reactive architectures:** Kubernetes controllers use etcd watches to react to state changes, forming a reconciliation loop

## Cross-System Comparison

| Property | Dynamo | Spanner | Kafka | CockroachDB | Cassandra | etcd |
|----------|--------|---------|-------|-------------|-----------|------|
| Primary goal | Availability | Consistency | Throughput | Consistency (no special HW) | Tuneable | Coordination |
| Partitioning | Hash (consistent) | Range (auto-split) | Hash (by key) | Range (auto-split) | Hash (vnodes) | None (small dataset) |
| Replication | Leaderless | Paxos/shard | Leader/partition | Raft/range | Leaderless | Raft |
| Consistency | Eventual | External | Per-partition linear | Serializable | Tuneable | Linearizable |
| Transactions | None (single-key) | Distributed ACID | Per-partition | Distributed ACID | Per-partition (LWT) | Per-key |
| Conflict resolution | Vector clocks + client | Locks + TrueTime | N/A (append-only) | MVCC + timestamps | LWW | N/A (leader-based) |
| Scale target | 100s of nodes | 1000s of nodes | 1000s of brokers | 100s of nodes | 1000s of nodes | 3–5 nodes |

## Key Takeaways

- **Dynamo** prioritises availability above all else — sloppy quorums, vector clocks, and client-side conflict resolution enable an always-writable system
- **Spanner** achieves the strongest consistency (external consistency) globally by investing in specialised hardware (TrueTime) and accepting higher write latency (commit-wait)
- **Kafka** treats the ordered log as the fundamental abstraction — per-partition total ordering with ISR-based replication provides both durability and throughput
- **CockroachDB** replicates Spanner's design using commodity hardware — HLC approximates TrueTime, Raft replaces Paxos, and parallel commits optimise 2PC
- **Cassandra** offers tunable consistency per query — the application decides the tradeoff for each operation, from ONE (fast, weak) to ALL (slow, strong)
- **etcd** runs a small Raft cluster providing linearizable reads and writes — purpose-built for coordination, not general-purpose storage
- Every system makes explicit tradeoffs between consistency, availability, latency, and throughput — understanding these tradeoffs is the core skill of distributed systems engineering
