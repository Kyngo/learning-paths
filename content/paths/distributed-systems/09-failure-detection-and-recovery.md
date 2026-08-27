---
title: Failure Detection & Recovery
weight: 9
---

# Failure Detection & Recovery

In a distributed system, detecting whether a remote node has failed is fundamentally uncertain. You cannot distinguish between a crashed node, a slow node, and a partitioned network — all three look the same from the outside: no response. This section covers the mechanisms systems use to detect failures, maintain cluster membership, and recover data after failures.

> **Note:** Application-level resilience patterns (circuit breakers, retries, bulkheads) are covered in [System Design: Reliability & Resilience]({{< relref "/paths/system-design/09-reliability-resilience" >}}). This section focuses on infrastructure-level failure detection and data recovery mechanisms.

## The Fundamental Problem

The impossibility of perfect failure detection in asynchronous systems was established alongside the FLP result (Section 1). A failure detector has two properties that are in tension:

| Property | Definition | Perfect means... |
|----------|-----------|-----------------|
| **Completeness** | Every crashed node is eventually detected | No false negatives |
| **Accuracy** | No running node is falsely detected as crashed | No false positives |

In an asynchronous system, you cannot have both. Practical failure detectors trade accuracy for completeness — they may occasionally suspect a healthy node (false positive) but will always detect actual failures (eventually).

## Heartbeat-Based Detection

The simplest failure detection mechanism: nodes periodically send "I'm alive" messages to a monitor.

### Direct Heartbeats

```
Node A ────── heartbeat ──────► Monitor
              (every Δ)

Monitor:
    last_heartbeat[A] = now()

    if now() - last_heartbeat[A] > timeout:
        suspect(A)

Typical values:
    Δ (heartbeat interval) = 1–5 seconds
    timeout = 3–5 × Δ
```

### Problems with Fixed Timeouts

| Issue | Consequence |
|-------|------------|
| **Timeout too short** | False positives during GC pauses, network congestion, or high load |
| **Timeout too long** | Slow detection — failed nodes hold resources longer |
| **Network jitter** | Variable latency makes any fixed timeout suboptimal |
| **GC pauses** | Java/Go stop-the-world pauses can exceed seconds |
| **CPU saturation** | Busy nodes miss heartbeat deadlines despite being alive |

### All-to-All Heartbeats

In a cluster of N nodes, every node sends heartbeats to every other node. This decentralises monitoring but generates O(N²) messages per interval.

```
N=5 nodes, Δ=1s:
    Messages per second = 5 × 4 = 20
N=100 nodes:
    Messages per second = 100 × 99 = 9,900
N=1000 nodes:
    Messages per second = 1,000 × 999 ≈ 1,000,000  ← unsustainable
```

For clusters beyond ~50 nodes, all-to-all heartbeats are impractical. Gossip-based protocols solve this scaling problem.

## Phi Accrual Failure Detector

The **phi accrual failure detector** (Hayashibara et al., 2004) replaces the binary "alive/dead" decision with a continuous **suspicion level** (φ) that increases as the time since the last heartbeat grows.

### How It Works

```
1. Track the distribution of heartbeat inter-arrival times
   (maintain a sliding window of recent intervals)

2. When a heartbeat is late, compute φ:
   φ = -log10(P(heartbeat_delay ≥ observed_delay))

   Where P is computed from the observed distribution
   (typically modelled as a normal distribution)

3. Interpret φ:
   φ = 1  → 10% chance the node is alive (P = 0.1)
   φ = 2  → 1% chance the node is alive (P = 0.01)
   φ = 3  → 0.1% chance the node is alive (P = 0.001)

4. Declare failure when φ exceeds a configurable threshold
```

### Advantages Over Fixed Timeouts

| Property | Fixed timeout | Phi accrual |
|----------|-------------|-------------|
| Adapts to network conditions | ❌ | ✅ (learned distribution) |
| Adapts to individual nodes | ❌ | ✅ (per-node statistics) |
| Configurable precision | Coarse (timeout value) | Fine (φ threshold) |
| False positive rate | Unpredictable | Tunable via threshold |

### Where It's Used

- **Akka** (default failure detector for Akka Cluster)
- **Cassandra** (inter-node failure detection)
- **Erlang/OTP** (inspiration for node monitoring)

## Gossip Protocols

Gossip (epidemic) protocols spread information through a cluster the way rumours spread in a social network: each node periodically tells a random peer what it knows, and the peer does the same.

### Basic Gossip

```
Every Δ seconds, each node:
    1. Select a random peer
    2. Exchange state (send yours, receive theirs)
    3. Merge received state with local state

State propagation:
    Round 1: 1 node knows → tells 1 peer → 2 nodes know
    Round 2: 2 nodes tell 2 peers → ~4 nodes know
    Round 3: 4 nodes tell 4 peers → ~8 nodes know
    ...
    After O(log N) rounds: all N nodes know

    Convergence time: O(log N) × Δ
```

### Properties

| Property | Value |
|----------|-------|
| Message complexity per round | O(N) — each node contacts one peer |
| Convergence time | O(log N) rounds |
| Fault tolerance | Resilient to node and message failures |
| Consistency | Probabilistic (eventually all nodes converge) |
| Scalability | Excellent — O(N) messages regardless of cluster size |

### Gossip for Failure Detection

Nodes include heartbeat counters in their gossip state. If a node's counter hasn't incremented after several gossip rounds, it is suspected.

```
Gossip state per node:
    {
        "node-A": {heartbeat: 42, timestamp: 1693000000},
        "node-B": {heartbeat: 87, timestamp: 1693000001},
        "node-C": {heartbeat: 15, timestamp: 1692999800}  ← stale!
    }

If node-C's heartbeat hasn't increased in T_fail seconds:
    Mark node-C as SUSPECTED

If still suspected after T_cleanup seconds:
    Mark node-C as DEAD, remove from membership
```

## SWIM: Scalable Weakly-Consistent Infection-Style Membership

SWIM (Das et al., 2002) is a membership protocol that combines failure detection with membership dissemination, designed for large clusters.

### SWIM Failure Detection

Instead of periodic heartbeats, SWIM uses a **probe-based** approach:

```
Every protocol period, node M_i:

1. Pick a random node M_j
2. Send ping(M_j)
3. If M_j responds (ack): M_j is alive

4. If M_j does NOT respond within timeout:
   a. Pick k random nodes (M_a, M_b, ... M_k)
   b. Send ping-req(M_j) to each
   c. They ping M_j on behalf of M_i
   d. If any receives an ack: M_j is alive
   e. If none receive an ack: mark M_j as SUSPECTED

    M_i ──── ping ────► M_j  (no response)

    M_i ── ping-req ──► M_a ── ping ──► M_j
    M_i ── ping-req ──► M_b ── ping ──► M_j
                                         │
    If all fail:    M_j is SUSPECTED ◄───┘
```

### Why Indirect Probes?

Direct probes can fail due to **asymmetric network issues** — M_i's link to M_j may be broken, but M_a's link to M_j may be fine. Indirect probes route around partial network failures.

### SWIM Dissemination

Membership changes (join, leave, suspect, confirm) are piggybacked on SWIM protocol messages rather than sent as separate gossip. This is called **infection-style dissemination** — information spreads as a side effect of failure detection.

```
Ping message:
    {
        type: "ping",
        membership_updates: [
            {node: "C", status: "suspected", incarnation: 5},
            {node: "D", status: "alive", incarnation: 12}
        ]
    }
```

### SWIM Properties

| Property | Value |
|----------|-------|
| Detection time | O(log N) protocol periods in expectation |
| Message load per node | O(1) per period (constant, regardless of cluster size) |
| False positive probability | Tunable via k (number of indirect probes) |
| Dissemination | O(log N) rounds (epidemic) |

SWIM is used by **HashiCorp Serf** (which powers Consul's gossip layer) and **Memberlist** (Go library).

## Anti-Entropy Mechanisms

After a failure and recovery, replicas must be brought back into sync. Several mechanisms work together.

### Read Repair

When a client reads from multiple replicas and detects a discrepancy, it writes the latest value back to the stale replica.

```
Client reads key "x" from 3 replicas:
    Replica A: x = 42 (version 5)
    Replica B: x = 42 (version 5)
    Replica C: x = 37 (version 3)   ← stale

Client writes x = 42 (version 5) back to Replica C
```

**Pro:** repairs are immediate and targeted.
**Con:** only repairs data that is actually read — rarely-accessed data may remain stale indefinitely.

### Hinted Handoff

When a replica is temporarily down, a **hint** is stored on another node. When the target recovers, the hint is forwarded to it.

```
Write to replicas {A, B, C}:
    A: ✓
    B: ✓
    C: ✗ (down)

Store hint on D: "When C recovers, send it (key=x, value=42, version=5)"

C recovers → D sends the hint → C is up to date
D deletes the hint
```

**Pro:** ensures writes reach their intended replicas after recovery.
**Con:** hints consume storage; if D also fails, hints are lost.

### Full Anti-Entropy (Merkle Tree Repair)

Periodically, replicas compare their data using **Merkle trees** (Section 8) and transfer any differences.

```
Schedule: every N minutes (e.g., hourly)

Replica A and Replica B:
    1. Exchange Merkle tree root hashes
    2. If roots match: done (data is identical)
    3. If roots differ: traverse tree to find differing leaves
    4. Transfer differing data blocks
    5. Both replicas are now in sync

Cost: O(log N) comparisons to find differences
```

### Comparison of Recovery Mechanisms

| Mechanism | When it runs | What it repairs | Latency | Completeness |
|-----------|-------------|----------------|---------|-------------|
| Read repair | On read | Only read data | Immediate | Low (unread data stays stale) |
| Hinted handoff | On recovery | Data written during downtime | Minutes | Medium (hints can be lost) |
| Anti-entropy | Periodic | All data | Hours | High (full comparison) |

Most systems use all three in combination. Read repair handles the hot path, hinted handoff catches recent writes, and anti-entropy is the safety net.

## Failure Detection in Practice

| System | Failure detector | Membership protocol | Recovery |
|--------|-----------------|--------------------|---------| 
| Cassandra | Phi accrual | Gossip | Read repair + anti-entropy + hinted handoff |
| DynamoDB | Gossip-based | Gossip | Merkle tree anti-entropy + hinted handoff |
| Consul/Serf | SWIM | SWIM (Memberlist) | Raft for consensus state |
| Kubernetes | Kubelet heartbeats | etcd-backed | Pod rescheduling |
| Akka Cluster | Phi accrual | Gossip | Application-level |
| ZooKeeper | Session timeouts | ZAB-managed | Leader-based recovery |

## Key Takeaways

- Perfect failure detection is impossible in asynchronous systems — every practical detector trades accuracy (false positives) for completeness (detecting all failures)
- Fixed heartbeat timeouts are simple but poorly adapted to variable network conditions and GC pauses
- The phi accrual failure detector provides a continuous suspicion level that adapts to observed network behaviour — used by Cassandra and Akka
- Gossip protocols scale failure detection and membership dissemination to large clusters with O(N) messages per round and O(log N) convergence time
- SWIM combines probe-based failure detection with epidemic dissemination — constant message load per node, used by HashiCorp Consul
- Recovery uses three complementary mechanisms: read repair (immediate, partial), hinted handoff (targeted, recent writes), and anti-entropy with Merkle trees (comprehensive, periodic)
- Production systems combine multiple mechanisms — no single approach handles all failure and recovery scenarios
