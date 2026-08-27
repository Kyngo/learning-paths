---
title: Consensus Algorithms
weight: 4
---

# Consensus Algorithms

Consensus is the problem of getting multiple nodes to agree on a single value, even when some nodes crash or messages are lost. It is the foundation of replicated state machines, leader election, distributed locks, and total order broadcast.

The FLP impossibility result (Section 1) proves that deterministic consensus is impossible in a purely asynchronous system. Practical algorithms sidestep this by assuming **partial synchrony** — the system is eventually timely enough for progress, even if delays occur.

## The Consensus Problem

### Formal Definition

A set of processes proposes values and must agree on one:

| Property | Requirement |
|----------|------------|
| **Validity** | The decided value was proposed by some process |
| **Agreement** | No two correct processes decide differently |
| **Termination** | Every correct process eventually decides |
| **Integrity** | Each process decides at most once |

Safety (validity + agreement + integrity) must hold **always**. Termination (liveness) is guaranteed only under partial synchrony — during periods of asynchrony, the algorithm may stall but will never produce a wrong answer.

## Paxos

Paxos, proposed by Leslie Lamport in 1989 (published 1998), is the foundational consensus algorithm. It is famously difficult to understand — Lamport's original paper described it as a protocol used by a fictional Greek parliament — but the core idea is elegant.

### Roles

| Role | Responsibility |
|------|---------------|
| **Proposer** | Proposes a value and drives the protocol |
| **Acceptor** | Votes on proposals; stores accepted values durably |
| **Learner** | Learns the decided value (often the same nodes as acceptors) |

A single node can play multiple roles. A cluster of 2F+1 acceptors tolerates F failures.

### Single-Decree Paxos (One Value)

The protocol has two phases:

```
Phase 1: PREPARE
─────────────────
Proposer                                    Acceptors
   │                                           │
   │── Prepare(n) ──────────────────────────►  │  "I want to propose with ballot n"
   │                                           │
   │◄── Promise(n, accepted_value?) ─────────  │  "I won't accept ballots < n"
   │                                           │   (returns any previously accepted value)

Phase 2: ACCEPT
─────────────────
   │                                           │
   │── Accept(n, value) ────────────────────►  │  "Accept this value with ballot n"
   │                                           │
   │◄── Accepted(n) ────────────────────────   │  "Accepted" (if no higher ballot seen)
   │                                           │
   └── value is decided when majority accepts ─┘
```

### Pseudocode

```
PROPOSER:
    choose ballot number n (unique, monotonically increasing)

    // Phase 1
    send Prepare(n) to majority of acceptors
    wait for Promise responses from majority
    if any acceptor already accepted a value:
        use the value from the highest-numbered ballot
    else:
        use proposer's own value

    // Phase 2
    send Accept(n, value) to majority of acceptors
    wait for Accepted from majority → value is decided

ACCEPTOR:
    state: highest_promised_ballot, accepted_ballot, accepted_value

    on Prepare(n):
        if n > highest_promised_ballot:
            highest_promised_ballot = n
            reply Promise(n, accepted_ballot, accepted_value)
        else:
            reject (or ignore)

    on Accept(n, value):
        if n >= highest_promised_ballot:
            accepted_ballot = n
            accepted_value = value
            reply Accepted(n)
        else:
            reject
```

### Why Paxos Is Correct

The key insight: if a value `v` has been accepted by a majority at ballot `n`, then **any future proposal at ballot > n will also propose `v`**. This is because the proposer in Phase 1 must consult a majority, and any majority overlaps with the majority that accepted `v`.

### Liveness Concern: Duelling Proposers

```
Proposer A: Prepare(1) → accepted by majority
Proposer B: Prepare(2) → accepted by majority (overrides A's promise)
Proposer A: Accept(1, v) → REJECTED (ballot too low)
Proposer A: Prepare(3) → accepted by majority (overrides B's promise)
Proposer B: Accept(2, v) → REJECTED
... infinite loop, no progress
```

Solution: elect a **distinguished proposer** (leader) so only one proposer is active at a time. This is an optimisation for liveness, not safety.

### Multi-Paxos

Single-decree Paxos decides **one** value. Real systems need to decide a **sequence** of values (a replicated log). Multi-Paxos optimises for this:

1. A stable leader is elected
2. The leader skips Phase 1 for subsequent slots (it already holds promises)
3. Each log entry requires only Phase 2 (one round trip)

```
Slot 1: Full Paxos (Phase 1 + Phase 2)  ← leader election
Slot 2: Phase 2 only (1 RTT)            ← leader reuses ballot
Slot 3: Phase 2 only (1 RTT)
...
Leader fails → new leader runs full Paxos to take over
```

## Raft

Raft (Ongaro & Ousterhout, 2014) was designed explicitly to be **understandable**. It solves the same problem as Multi-Paxos but decomposes it into three relatively independent subproblems.

### Raft Subproblems

| Subproblem | Mechanism |
|-----------|-----------|
| Leader election | Randomised timeouts, term numbers |
| Log replication | Leader appends entries, followers replicate |
| Safety | Election restriction guarantees log completeness |

### State

Each node is in one of three states:

```
                    timeout, start election
            ┌──────────────────────────────────┐
            ▼                                  │
        Candidate ──── receives majority ──► Leader
            │              votes               │
            │                                  │
            │  discovers leader or             │ discovers higher
            │  higher term                     │ term
            ▼                                  ▼
        Follower ◄─────────────────────── Follower
```

### Leader Election

```
1. Follower times out (no heartbeat from leader)
2. Increments term, transitions to Candidate
3. Votes for itself
4. Sends RequestVote to all other nodes
5. Wins if it receives votes from a majority
6. If another node has a higher term → step down to Follower

Election timeout: randomised (e.g., 150–300 ms) to avoid split votes
```

### Log Replication

```
Client → Leader: "write x=5"

Leader:
    1. Append entry to local log: (term=3, index=7, cmd="x=5")
    2. Send AppendEntries(entries, prevLogIndex, prevLogTerm) to all followers
    3. Wait for majority acknowledgment
    4. Commit the entry (advance commitIndex)
    5. Apply to state machine
    6. Respond to client

Followers:
    1. Check prevLogIndex and prevLogTerm match local log
    2. If match: append entry, acknowledge
    3. If mismatch: reject → leader retries with earlier entries
```

```
Leader log:    [1:a] [1:b] [2:c] [3:d] [3:e]
                                        ↑ commitIndex = 5

Follower A:    [1:a] [1:b] [2:c] [3:d] [3:e]  ✓ up to date
Follower B:    [1:a] [1:b] [2:c] [3:d]         ✓ one behind (will catch up)
Follower C:    [1:a] [1:b]                       ✗ far behind (leader backtracks)
```

### Safety: Election Restriction

Raft guarantees that a newly elected leader has **all committed entries** by requiring:

> A candidate can only win an election if its log is **at least as up-to-date** as the logs of a majority of nodes.

"At least as up-to-date" means: last entry has a higher term, or same term with equal or greater index.

This eliminates the need for the new leader to "learn" committed entries from other nodes — it already has them.

### Raft vs Multi-Paxos

| Aspect | Raft | Multi-Paxos |
|--------|------|-------------|
| Understandability | Designed for clarity | Notoriously difficult |
| Leader | Strong leader required | Optional (but used in practice) |
| Log structure | Entries committed in order | Gaps allowed, filled later |
| Leader election | Randomised timeout | Separate protocol (flexible) |
| Log divergence | Leader overwrites conflicting entries | More complex reconciliation |
| Production systems | etcd, CockroachDB, TiKV, Consul | Chubby, Spanner (internal variant) |

## Viewstamped Replication (VR)

Viewstamped Replication (Oki & Liskov, 1988) predates Paxos and solves the same problem with a different decomposition.

### Key Concepts

| Concept | Description |
|---------|------------|
| **View** | A numbered configuration with a designated primary |
| **View change** | Leader election equivalent — triggered by timeout |
| **Normal operation** | Primary assigns order numbers and replicates to backups |

### Normal Operation

```
Client → Primary: request
Primary:
    1. Assign op-number
    2. Send Prepare(view, op-number, request) to backups
    3. Wait for f PrepareOk responses (f = number of tolerated failures)
    4. Commit and reply to client
    5. Send Commit(op-number) to backups
```

VR is structurally similar to Raft but was developed independently. It influenced many practical systems, including the design of Raft itself.

## ZAB (ZooKeeper Atomic Broadcast)

ZAB is the consensus protocol used by Apache ZooKeeper. It is optimised for the primary-backup pattern with ordered broadcasts.

### Phases

| Phase | Purpose |
|-------|---------|
| **Discovery** | New leader learns the latest state from a quorum |
| **Synchronisation** | Leader brings all followers up to date |
| **Broadcast** | Leader processes client requests and replicates |

### Key Difference from Raft

ZAB separates **recovery** (discovery + synchronisation) from **broadcast** (normal operation). It also uses **transaction IDs** (zxid) with an epoch component, similar to Raft's term numbers.

```
zxid = (epoch, counter)
      epoch increments on leader change
      counter increments per transaction within an epoch
```

## Practical Comparison

| Property | Paxos | Multi-Paxos | Raft | VR | ZAB |
|----------|-------|-------------|------|----|-----|
| Tolerates F failures in 2F+1 nodes | ✅ | ✅ | ✅ | ✅ | ✅ |
| Requires stable leader | ❌ | Preferred | ✅ | ✅ | ✅ |
| Log gaps allowed | N/A | ✅ | ❌ | ❌ | ❌ |
| Designed for understandability | ❌ | ❌ | ✅ | Medium | Medium |
| Production use | Limited | Google (internal) | etcd, CockroachDB, TiKV | Harp (research) | ZooKeeper |
| Reconfiguration built-in | ❌ | Varies | ✅ (joint consensus) | ✅ (view changes) | ✅ |

### Choosing a Consensus Algorithm

In practice, **Raft** is the default choice for new systems:

- Well-documented, well-tested, widely implemented
- Strong open-source implementations (etcd, HashiCorp Raft library)
- Easier to implement correctly than Paxos

Use ZAB if you are running ZooKeeper (you don't choose ZAB independently). Use Paxos variants only if you have specific requirements that Raft cannot meet (e.g., leaderless consensus for lower latency in certain topologies).

## Key Takeaways

- Consensus is the problem of getting nodes to agree on a value despite crashes and message loss — it underpins replicated state machines, leader election, and coordination
- Paxos is the foundational algorithm: two-phase protocol (prepare/accept) that guarantees safety always and liveness under partial synchrony
- Multi-Paxos optimises Paxos for a sequence of decisions by amortising Phase 1 across a stable leader's tenure
- Raft decomposes consensus into leader election, log replication, and safety — designed for understandability and widely adopted in production
- Raft's election restriction guarantees that any elected leader already has all committed log entries
- VR and ZAB solve the same problem with different decompositions — VR influenced Raft's design, ZAB powers ZooKeeper
- All these algorithms require a majority quorum (2F+1 nodes to tolerate F failures) and assume crash-recovery with partial synchrony
