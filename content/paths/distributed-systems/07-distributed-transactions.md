---
title: Distributed Transactions
weight: 7
---

# Distributed Transactions

A distributed transaction spans multiple nodes, partitions, or services. It must either commit on **all** participants or abort on **all** — partial completion is not acceptable. This is far harder than single-node transactions because any participant can crash, and the network can partition at any point during the protocol.

This section covers the classical protocols (2PC, 3PC), modern approaches (deterministic databases, Spanner's TrueTime), and the coordination mechanisms that make them work.

> **Note:** The Saga pattern for long-running cross-service transactions is covered in [System Design: Microservices]({{< relref "/paths/system-design/07-microservices" >}}) and [Software Architecture: Event-Driven Architecture]({{< relref "/paths/software-architecture/05-event-driven-architecture" >}}).

## Two-Phase Commit (2PC)

Two-Phase Commit is the classical protocol for atomic distributed transactions. It uses a **coordinator** (transaction manager) and multiple **participants** (resource managers).

### Protocol

```
Phase 1: PREPARE (voting)
──────────────────────────
Coordinator                          Participants
    │                                    │
    │── Prepare ───────────────────────► │  "Can you commit?"
    │                                    │
    │◄── Vote YES/NO ──────────────────  │  Each participant:
    │                                    │   - Acquires locks
    │                                    │   - Writes to WAL
    │                                    │   - Votes YES (can commit) or NO (cannot)

Phase 2: COMMIT/ABORT (decision)
─────────────────────────────────
    │                                    │
    │  If ALL voted YES:                 │
    │── Commit ────────────────────────► │  "Commit the transaction"
    │                                    │
    │  If ANY voted NO:                  │
    │── Abort ─────────────────────────► │  "Abort the transaction"
    │                                    │
    │◄── ACK ──────────────────────────  │  Participant confirms
```

### Pseudocode

```
COORDINATOR:
    // Phase 1
    for each participant:
        send Prepare(txn_id)
    wait for all votes (with timeout)

    if all votes == YES:
        write COMMIT to durable log  ← point of no return
        // Phase 2
        for each participant:
            send Commit(txn_id)
    else:
        write ABORT to durable log
        for each participant:
            send Abort(txn_id)

PARTICIPANT:
    on Prepare(txn_id):
        if can_commit(txn_id):
            write PREPARED to WAL
            acquire locks
            reply YES
        else:
            reply NO

    on Commit(txn_id):
        apply transaction
        release locks
        reply ACK

    on Abort(txn_id):
        rollback transaction
        release locks
        reply ACK
```

### The Blocking Problem

2PC's critical weakness: if the coordinator crashes **after sending Prepare but before sending the decision**, participants that voted YES are **stuck**. They cannot commit (they don't know if everyone voted YES) and cannot abort (the coordinator may have decided to commit).

```
Coordinator: Prepare → [CRASH]

Participant A: voted YES → holding locks → waiting forever
Participant B: voted YES → holding locks → waiting forever

Neither can safely proceed. This state persists until the coordinator recovers.
```

This is the **blocking problem** — the fundamental limitation of 2PC. Participants hold locks indefinitely, blocking other transactions.

### Recovery

When the coordinator recovers, it reads its durable log:

| Log state | Recovery action |
|-----------|----------------|
| No COMMIT or ABORT record | Abort the transaction |
| COMMIT record | Resend Commit to all participants |
| ABORT record | Resend Abort to all participants |

Participants that crashed recover by reading their WAL:

| WAL state | Recovery action |
|-----------|----------------|
| No PREPARED record | Transaction was not prepared; abort locally |
| PREPARED record, no decision | Contact coordinator for the decision (may block) |
| COMMITTED or ABORTED | Apply the decision |

## Three-Phase Commit (3PC)

3PC was designed to eliminate the blocking problem by adding a **pre-commit** phase.

### Protocol

```
Phase 1: CAN COMMIT?
    Coordinator → Participants: "Can you commit?"
    Participants → Coordinator: YES/NO

Phase 2: PRE-COMMIT
    Coordinator → Participants: "Prepare to commit" (if all said YES)
    Participants → Coordinator: ACK

Phase 3: DO COMMIT
    Coordinator → Participants: "Commit now"
    Participants → Coordinator: ACK
```

### Why 3PC Doesn't Block

If the coordinator crashes after Phase 2, participants know that **all participants said YES** (otherwise Phase 2 wouldn't have been reached). They can elect a new coordinator and proceed to commit.

### Why 3PC Is Not Used in Practice

| Issue | Impact |
|-------|--------|
| Assumes no network partitions | In a partition, one side may commit while the other aborts |
| Extra round trip | Higher latency (3 RTTs vs 2 RTTs) |
| Complexity | More states to manage, more failure modes |
| Network partitions break safety | The assumption of no partitions is unrealistic |

**Verdict:** 3PC solves the blocking problem but introduces a safety violation under network partitions. Since partitions are inevitable in real systems, 3PC is almost never used. Modern systems use Paxos/Raft-based commit protocols instead.

## Distributed Locking

When distributed transactions access shared resources, **distributed locks** prevent concurrent access. The most common approach is using a separate lock manager backed by consensus.

### Lock Manager Architecture

```
Client A                                      Lock Service
   │                                          (e.g., etcd, ZooKeeper)
   │── Acquire lock("resource-42") ────────►  │
   │                                          │  Creates ephemeral key
   │◄── Lock granted (lease TTL=30s) ────────  │  with TTL
   │                                          │
   │  ... perform operation ...               │
   │                                          │
   │── Release lock("resource-42") ──────────► │
   │                                          │  Deletes key
```

### Fencing Tokens

A lock alone is not sufficient — a client may believe it holds a lock after the lease expires (e.g., due to a GC pause). **Fencing tokens** solve this by associating a monotonically increasing token with each lock grant.

```
Client A acquires lock → fencing token = 34
Client A pauses (GC) → lock expires
Client B acquires lock → fencing token = 35
Client A wakes up, sends write with token 34
Storage: "token 34 < current token 35 → REJECT"
```

| Without fencing | With fencing |
|----------------|-------------|
| Client A's stale write succeeds | Client A's stale write is rejected |
| Data corruption | Correctness preserved |

### Redlock (and Its Controversy)

Redis-based distributed locking using the Redlock algorithm proposes acquiring locks on a majority of independent Redis instances. However, Martin Kleppmann's analysis ("How to do distributed locking", 2016) demonstrated that Redlock is unsafe because Redis lacks consensus and persistence guarantees.

**Recommendation:** use a consensus-backed lock service (etcd, ZooKeeper, Consul) for correctness-critical locking. Redis locks are acceptable only when approximate mutual exclusion is sufficient (e.g., deduplication, rate limiting).

## Deterministic Databases (Calvin)

Calvin (Thomson et al., 2012) takes a radically different approach: instead of coordinating **during** transaction execution, it agrees on the **order of transactions** upfront using consensus, then each replica executes them deterministically.

### Architecture

```
Step 1: Sequencer
    Batches incoming transactions and assigns a global order
    using Paxos/Raft across replicas

Step 2: Scheduler
    Each replica receives the same ordered batch
    Analyses read/write sets to determine dependencies
    Executes non-conflicting transactions in parallel

Step 3: Execution
    Each replica executes transactions in the agreed order
    Deterministic execution → all replicas reach the same state
    No 2PC needed — the order was already agreed upon
```

### Advantages

| Property | Calvin | Traditional (2PC) |
|----------|--------|-------------------|
| Coordination during execution | None | Per-transaction |
| Cross-partition transactions | Cheap (predetermined order) | Expensive (2PC) |
| Replica consistency | By construction (same input, same output) | Requires additional protocols |
| Recovery | Replay the log | Complex coordinator recovery |

### Limitations

- **Read/write sets must be known in advance** — interactive transactions (where the next query depends on the previous result) require a reconnaissance phase
- **Throughput ceiling** — the sequencer is a bottleneck for very high write rates
- **All transactions go through consensus** — even single-partition ones

Calvin's ideas are used in **FaunaDB** and influence the design of **CockroachDB**'s transaction layer.

## Google Spanner and TrueTime

Spanner (Corbett et al., 2012) achieves **external consistency** (linearizable + serializable) globally by using **TrueTime** — a clock API that provides bounded uncertainty.

### TrueTime API

```
TrueTime.now() → [earliest, latest]

Returns an interval: the true time is guaranteed to be within [earliest, latest].
Typical uncertainty: ε ≈ 1–7 ms (GPS + atomic clocks in each datacenter).
```

### Commit-Wait Protocol

When a transaction commits, Spanner **waits** until the uncertainty interval has passed. This ensures that the commit timestamp is in the past for all nodes.

```
Transaction T commits at timestamp s.
Spanner waits until TrueTime.now().earliest > s.
Wait duration ≈ 2ε (typically 2–14 ms).

After the wait:
  Every node's clock is past s → T is visible to all.
  Any subsequent transaction gets a timestamp > s.
  → External consistency: if T1 commits before T2 starts,
     T1's timestamp < T2's timestamp. Always.
```

### Why This Works

| Component | Purpose |
|-----------|---------|
| GPS receivers + atomic clocks | Keep ε small (1–7 ms) |
| Bounded uncertainty interval | Know the worst case |
| Commit-wait | Ensure timestamp ordering matches real-time ordering |
| Paxos per shard | Replicate each shard's data |
| 2PC across shards | Atomicity for multi-shard transactions |

### Cost of TrueTime

- **Specialised hardware** — GPS receivers and atomic clocks in every datacenter
- **Commit latency floor** — every write waits 2ε (typically 4–14 ms)
- **Not available outside Google** — CockroachDB approximates this with HLC (Section 2)

## Transaction Coordinators in Practice

Real systems implement distributed transaction coordination as infrastructure:

| System | Coordinator | Protocol |
|--------|------------|----------|
| PostgreSQL (FDW) | Postgres with `postgres_fdw` | 2PC (`PREPARE TRANSACTION`) |
| MySQL (XA) | Application or MySQL | XA (variant of 2PC) |
| Google Spanner | Per-transaction Paxos leader | 2PC + TrueTime |
| CockroachDB | Leaseholder of the first range | Parallel commits (optimised 2PC) |
| Amazon DynamoDB (transactions) | Internal transaction coordinator | Timestamp ordering |
| Kafka (exactly-once) | Transaction coordinator broker | 2PC between producers and brokers |

### CockroachDB's Parallel Commits

CockroachDB optimises 2PC by pipelining: the coordinator writes the transaction record and the intent writes **in parallel**, then a single round of consensus commits everything. This reduces commit latency from 2 consensus rounds to 1 in the common case.

```
Traditional 2PC:
    Round 1: Prepare (consensus) → Round 2: Commit (consensus)
    Latency: 2 × consensus round

CockroachDB Parallel Commits:
    Round 1: Write intents + transaction record (parallel consensus)
    Latency: 1 × consensus round (common case)
```

## Key Takeaways

- Two-Phase Commit (2PC) ensures atomicity across nodes but **blocks** if the coordinator crashes after Prepare — participants hold locks indefinitely
- Three-Phase Commit (3PC) solves blocking but is unsafe under network partitions — it is not used in practice
- Distributed locks require fencing tokens to prevent stale clients from corrupting data — use consensus-backed services (etcd, ZooKeeper), not Redis, for correctness-critical locking
- Calvin (deterministic databases) avoids per-transaction coordination by agreeing on transaction order upfront via consensus — replicas execute deterministically and converge by construction
- Spanner uses TrueTime (GPS + atomic clocks) to provide external consistency globally — the commit-wait protocol ensures timestamp ordering matches real-time ordering
- Modern databases (CockroachDB, Spanner) optimise 2PC with pipelining and parallel consensus to reduce latency
- The Saga pattern (covered elsewhere) is the alternative when distributed ACID transactions are too expensive — it trades atomicity for availability using compensating actions
