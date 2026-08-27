---
title: CRDTs & Eventual Consistency
weight: 8
---

# CRDTs & Eventual Consistency

Conflict-free Replicated Data Types (CRDTs) are data structures that can be replicated across multiple nodes, modified independently and concurrently, and merged automatically — **always converging** to the same state without coordination. They provide **strong eventual consistency** (Section 3): any two replicas that have received the same set of updates are in the same state, immediately.

This section covers the theory behind CRDTs, the most important CRDT types, and the anti-entropy mechanisms that propagate updates between replicas.

## Why CRDTs?

Traditional approaches to concurrent updates require either:

- **Coordination** (locks, consensus) — high latency, unavailable during partitions
- **Last-writer-wins** — simple, but silently loses data
- **Application-level conflict resolution** — correct, but complex and error-prone

CRDTs eliminate the need for all three by making **merge a mathematical property** of the data structure itself. If two replicas receive the same updates in any order, they converge to the same state — guaranteed by the data type's algebraic properties.

### Requirements for a CRDT

A data type is a CRDT if its merge operation forms a **join semilattice**:

| Property | Requirement | Why it matters |
|----------|------------|----------------|
| **Commutativity** | merge(a, b) = merge(b, a) | Order of receiving updates doesn't matter |
| **Associativity** | merge(a, merge(b, c)) = merge(merge(a, b), c) | Grouping of merges doesn't matter |
| **Idempotency** | merge(a, a) = a | Duplicate delivery is harmless |

## State-Based vs Operation-Based CRDTs

CRDTs come in two families that differ in **what** is transmitted between replicas.

### State-Based (CvRDT — Convergent)

Replicas periodically send their **full state** to other replicas. The receiving replica merges the incoming state with its own using a merge function.

```
Replica A state: {x: 3, y: 5}
Replica B state: {x: 4, y: 2}

Merge (element-wise max): {x: 4, y: 5}

Both replicas converge to {x: 4, y: 5} regardless of merge order.
```

**Pro:** tolerates message loss, reordering, and duplication (idempotent merge).
**Con:** transmitting full state can be expensive for large data structures.

### Operation-Based (CmRDT — Commutative)

Replicas broadcast **operations** (deltas) to other replicas. Each replica applies the operations to its local state.

```
Replica A: increment(x)   → broadcasts "increment(x)"
Replica B: increment(x)   → broadcasts "increment(x)"

Both replicas apply both increments → x = original + 2
```

**Pro:** transmits only deltas (smaller messages).
**Con:** requires **reliable causal broadcast** — every operation must be delivered exactly once in causal order.

### Comparison

| Property | State-based (CvRDT) | Operation-based (CmRDT) |
|----------|--------------------|-----------------------|
| Message size | Full state (large) | Operation (small) |
| Delivery requirement | At-least-once (any channel) | Exactly-once, causal order |
| Merge complexity | Must be idempotent | Must be commutative |
| Network tolerance | Very high (works with gossip) | Requires reliable broadcast |

In practice, **delta-state CRDTs** combine the best of both: they transmit only the state changes (deltas) since the last sync, with the robustness of state-based merge.

## Common CRDT Types

### G-Counter (Grow-Only Counter)

A counter that can only be incremented. Each replica maintains its own count; the total is the sum of all counts.

```
Structure: vector of per-replica counts
    {A: 0, B: 0, C: 0}

Replica A increments twice:   {A: 2, B: 0, C: 0}
Replica B increments once:    {A: 0, B: 1, C: 0}
Replica C increments three:   {A: 0, B: 0, C: 3}

Merge (element-wise max):
    {A: 2, B: 1, C: 3}

Value = sum(2 + 1 + 3) = 6
```

```
merge(X, Y):
    for each replica r:
        result[r] = max(X[r], Y[r])

value(counter):
    return sum(counter.values())

increment(counter, replica_id):
    counter[replica_id] += 1
```

### PN-Counter (Positive-Negative Counter)

A counter that supports both increment and decrement, built from two G-Counters.

```
Structure: P (positive G-Counter) + N (negative G-Counter)

increment(replica_id):  P[replica_id] += 1
decrement(replica_id):  N[replica_id] += 1
value():                sum(P) - sum(N)

Example:
    P = {A: 5, B: 3}  → sum = 8
    N = {A: 2, B: 1}  → sum = 3
    Value = 8 - 3 = 5

Merge:
    merge P using G-Counter merge (element-wise max)
    merge N using G-Counter merge (element-wise max)
```

### LWW-Register (Last-Writer-Wins Register)

A register that resolves concurrent writes by keeping the one with the highest timestamp.

```
Structure: (value, timestamp)

write(v, t):
    if t > current_timestamp:
        value = v
        timestamp = t

merge(A, B):
    return A if A.timestamp > B.timestamp else B

Example:
    Replica A: (value="alice", ts=100)
    Replica B: (value="bob", ts=105)
    Merge → (value="bob", ts=105)
```

**Warning:** LWW-Register is technically a CRDT (merge is commutative, associative, idempotent), but it **silently discards writes**. It is only appropriate when losing concurrent updates is acceptable.

### OR-Set (Observed-Remove Set)

A set that supports both add and remove, resolving the "add/remove concurrency" problem. The core idea: each added element carries a **unique tag**. Remove only removes tags that the remover has **observed**.

```
add("apple"):    generate unique tag t1
                 state = {("apple", t1)}

add("apple"):    generate unique tag t2  (different replica)
                 state = {("apple", t2)}

Merge:           {("apple", t1), ("apple", t2)}

remove("apple"): removes only tags known to this replica
                 if this replica has seen t1 but not t2:
                 removes ("apple", t1)
                 result: {("apple", t2)} → "apple" is still in the set
```

### Semantics Comparison

```
Scenario: Replica A adds "apple", Replica B concurrently removes "apple"

Add-wins (OR-Set): "apple" remains (add wins over concurrent remove)
Remove-wins:       "apple" is removed (remove wins)
LWW:               depends on timestamp (non-deterministic from user perspective)
```

### Summary of CRDT Types

| CRDT | Operations | Conflict resolution | Use case |
|------|-----------|-------------------|----------|
| G-Counter | Increment only | Sum of per-replica counts | Page views, likes |
| PN-Counter | Increment + decrement | Two G-Counters (P - N) | Inventory count, votes |
| LWW-Register | Write | Highest timestamp wins | Simple key-value cache |
| MV-Register | Write | Return all concurrent values | Shopping cart (app resolves) |
| G-Set | Add only | Union | Tag lists, follower sets |
| OR-Set | Add + remove | Unique tags, add-wins | Collaborative todo lists |
| LWW-Map | Set key/value | Per-key LWW | Configuration stores |
| RGA | Insert + delete at position | Unique position IDs | Collaborative text editing |

## Merkle Trees for Anti-Entropy

When replicas diverge (e.g., after a network partition), they need to **find and repair differences**. Comparing full datasets is expensive. Merkle trees make this efficient.

### Structure

A **Merkle tree** (hash tree) is a binary tree where:

- Each **leaf** contains the hash of a data block
- Each **internal node** contains the hash of its children's hashes
- The **root hash** summarises the entire dataset

```
                    Root: H(H12 + H34)
                   /                    \
            H12: H(H1+H2)          H34: H(H3+H4)
           /          \            /          \
      H1: H(data1)  H2: H(data2)  H3: H(data3)  H4: H(data4)
          |              |              |              |
        data1          data2          data3          data4
```

### Anti-Entropy Protocol

```
Replica A                              Replica B
    │                                      │
    │── Send root hash ──────────────────► │
    │                                      │  Compare root hashes
    │◄── Roots differ ────────────────────  │
    │                                      │
    │── Send left and right child hashes ─► │
    │                                      │  Left subtree matches,
    │◄── Right subtree differs ───────────  │  right subtree differs
    │                                      │
    │── Send right subtree children ──────► │
    │                                      │  Leaf H3 differs
    │◄── Send data3 ─────────────────────  │
    │                                      │
    │  Repair: update local data3          │

Comparisons needed: O(log N) instead of O(N)
```

### Where Merkle Trees Are Used

| System | Use |
|--------|-----|
| Cassandra | Anti-entropy repair between replicas |
| DynamoDB | Background consistency verification |
| IPFS | Content-addressable storage verification |
| Git | Object integrity (every commit is a Merkle DAG) |
| Bitcoin/Ethereum | Transaction verification in blocks |

## Practical CRDT Implementations

| System | CRDTs used | Purpose |
|--------|-----------|---------|
| Riak | Counters, Sets, Maps, Flags | General-purpose K/V store with conflict-free types |
| Redis (CRDTBs) | Counters, Sets, Strings | Active-active geo-replication |
| Automerge | JSON CRDT (registers, lists, maps) | Collaborative editing (local-first apps) |
| Yjs | Text, Array, Map, XML CRDTs | Real-time collaborative editing |
| Cassandra | Counters (PN-Counter) | Distributed counter columns |
| Phoenix (Elixir) | OR-Set, PN-Counter | Distributed presence tracking |

### Operational Considerations

| Concern | Mitigation |
|---------|-----------|
| **Metadata growth** | Tombstones and version vectors grow without bound — implement garbage collection (compaction) |
| **Tombstone accumulation** | Deleted elements leave markers — prune tombstones after a safe interval (e.g., GC grace period) |
| **Causality tracking** | OR-Sets track tags per element — use compact representations (dot-kernel) |
| **Debugging** | Convergence is guaranteed but the merged state can be surprising — invest in observability |

## Key Takeaways

- CRDTs are data structures whose merge operation is commutative, associative, and idempotent — any two replicas with the same updates converge to the same state without coordination
- State-based CRDTs transmit full state and work with unreliable channels; operation-based CRDTs transmit operations and require reliable causal delivery
- G-Counter and PN-Counter handle increment/decrement without conflicts; OR-Set handles add/remove with add-wins semantics; LWW-Register resolves by timestamp (with data loss)
- Merkle trees enable efficient anti-entropy by comparing hierarchical hashes — finding differences in O(log N) instead of O(N)
- CRDTs are used in production by Riak, Redis, Cassandra, Automerge, and Yjs — they are the foundation of conflict-free collaborative editing and geo-replicated databases
- CRDTs trade metadata overhead (version vectors, tombstones) for coordination freedom — garbage collection and compaction are essential operational concerns
