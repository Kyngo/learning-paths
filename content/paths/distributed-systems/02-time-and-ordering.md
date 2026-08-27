---
title: Time & Ordering
weight: 2
---

# Time & Ordering

In a single-node system, "what happened first?" has a straightforward answer — the CPU's instruction counter provides a total order. In a distributed system, this question becomes one of the hardest problems in computer science. There is no global clock, no shared counter, and no way to determine the exact order of events across nodes without communication.

This section covers the tools we use to establish ordering in distributed systems — from physical clocks (and why they fail) to logical clocks that capture causality without relying on time at all.

## Physical Clocks and Their Limitations

Every computer has a hardware clock, typically a quartz crystal oscillator disciplined by NTP (Network Time Protocol) or PTP (Precision Time Protocol). These clocks are useful but fundamentally unreliable for ordering distributed events.

### Clock Skew and Drift

| Term | Definition | Typical magnitude |
|------|-----------|-------------------|
| **Clock skew** | Difference between two clocks at a given instant | 1–100 ms across datacenter nodes |
| **Clock drift** | Rate at which a clock diverges from true time | 10–200 ppm (quartz), 1 ppm (TCXO) |
| **Clock jump** | Sudden correction applied by NTP | Can be seconds after long drift |

```
True time:     ──────────────────────────────────►

Node A clock:  ─────────────────────────────────► (drifts ahead by 50ms)

Node B clock:  ──────────────────────────────►     (drifts behind by 30ms)

                                               ↑
                              Event at "same instant" appears
                              80ms apart on A and B
```

### Why Physical Clocks Cannot Order Events

Consider two events:

1. Node A writes `x = 1` at timestamp `T_A = 100`
2. Node B writes `x = 2` at timestamp `T_B = 101`

If Node A's clock is 50 ms ahead of true time, the actual order might be reversed — Node B wrote first. Using physical timestamps to resolve conflicts (last-writer-wins) can silently lose data.

### NTP and Its Bounds

NTP synchronises clocks to within **1–10 ms** in a well-configured datacenter. But:

- NTP corrections can cause **time to jump backward** (unless `slew` mode is used, which limits correction rate)
- NTP accuracy degrades across WAN links (50–100 ms typical)
- Clock discipline can fail silently if the NTP server itself drifts

**Rule of thumb:** physical time is useful for human-readable timestamps and coarse ordering, but never for determining causality between events on different nodes.

## The Happens-Before Relation

Leslie Lamport's 1978 paper "Time, Clocks, and the Ordering of Events in a Distributed System" introduced the **happens-before** relation (→), which captures causal ordering without physical time.

### Definition

For events `a` and `b`, `a → b` (a happens before b) if:

1. **Same process:** `a` and `b` are events in the same process, and `a` occurs before `b` in program order
2. **Message passing:** `a` is the send of a message, and `b` is the receipt of that same message
3. **Transitivity:** if `a → b` and `b → c`, then `a → c`

If neither `a → b` nor `b → a`, the events are **concurrent** (written `a ‖ b`). Concurrent events have no causal relationship — they could have happened in either order.

```
Process P1:    a ─────────────── c ──────────── e
                    \                  ↑
                     \  message       / message
                      ↓              /
Process P2:          b ──────── d ─/

Ordering: a → b → d → c → e
Events a and d are NOT concurrent (a → b → d)
```

### Why This Matters

The happens-before relation tells us which events **could have influenced** other events. If `a → b`, then `a` might have caused `b`. If `a ‖ b`, there is no possible causal link, and any conflict between them must be resolved by the application.

## Lamport Clocks

Lamport clocks assign a single integer counter to each event, preserving the happens-before relation.

### Algorithm

```
Each process maintains a counter C, initially 0.

On local event:
    C = C + 1
    assign timestamp C to the event

On sending message:
    C = C + 1
    attach C to the message

On receiving message with timestamp T:
    C = max(C, T) + 1
    assign timestamp C to the receive event
```

### Example

```
P1:  C=1        C=2                    C=5
     (a)────────(b)────────────────────(e)
                  \                    ↑
                   \ msg(C=2)        / msg(C=4)
                    ↓               /
P2:          C=1   C=3        C=4
             (x)───(c)────────(d)──/
```

### Properties

| Property | Lamport clocks |
|----------|---------------|
| If `a → b` then `L(a) < L(b)` | ✅ Yes — this is guaranteed |
| If `L(a) < L(b)` then `a → b` | ❌ No — concurrent events can have different timestamps |
| Detect concurrency | ❌ Cannot distinguish "happens-before" from "concurrent" |

Lamport clocks give a **consistent total order** (break ties by process ID), but they cannot detect whether two events are causally related or concurrent.

## Vector Clocks

Vector clocks extend Lamport clocks to capture the **full causal history** of each event, enabling detection of concurrent events.

### Algorithm

```
Each process i maintains a vector V[1..N] (N = number of processes),
initially all zeros.

On local event at process i:
    V[i] = V[i] + 1

On sending message at process i:
    V[i] = V[i] + 1
    attach V to the message

On receiving message with vector T at process i:
    for each j in 1..N:
        V[j] = max(V[j], T[j])
    V[i] = V[i] + 1
```

### Example (3 processes)

```
P1: [1,0,0]    [2,0,0]             [3,2,2]
     (a)─────────(b)──────────────────(f)
                   \                  ↑
                    \ msg            / msg
                     ↓              /
P2:          [0,1,0] [2,2,0] [2,3,0]
              (c)──────(d)──────(e)──/
                                ↑
                               / msg
                              /
P3:                    [0,0,1] [0,0,2]
                        (g)──────(h)
```

### Comparing Vector Clocks

Given vectors `V_a` and `V_b`:

| Condition | Meaning |
|-----------|---------|
| `V_a[i] ≤ V_b[i]` for all `i`, and `V_a ≠ V_b` | `a → b` (a happened before b) |
| `V_b[i] ≤ V_a[i]` for all `i`, and `V_a ≠ V_b` | `b → a` (b happened before a) |
| Neither of the above | `a ‖ b` (concurrent) |

This is the critical advantage over Lamport clocks: vector clocks can **detect concurrency**.

### Practical Limitations

| Issue | Impact |
|-------|--------|
| **Size** | Vector grows with number of processes (O(N) per event) |
| **Dynamic membership** | Adding/removing nodes requires resizing all vectors |
| **Garbage collection** | Old entries cannot be pruned without losing causal information |

For systems with thousands of nodes, full vector clocks become impractical. Solutions include **dotted version vectors** and **bounded vector clocks**.

## Hybrid Logical Clocks (HLC)

Hybrid Logical Clocks (Kulkarni et al., 2014) combine physical time with logical counters to get the best of both worlds.

### Structure

An HLC timestamp has two components:

```
HLC = (physical_time, logical_counter)

- physical_time: the node's NTP-synchronised wall clock
- logical_counter: incremented when physical_time hasn't advanced
```

### Algorithm

```
On local event or send at process i:
    pt = physical_clock()
    if pt > hlc.physical:
        hlc.physical = pt
        hlc.logical = 0
    else:
        hlc.logical = hlc.logical + 1

On receiving message with timestamp (msg_pt, msg_l):
    pt = physical_clock()
    if pt > hlc.physical and pt > msg_pt:
        hlc.physical = pt
        hlc.logical = 0
    elif hlc.physical > msg_pt:
        hlc.logical = hlc.logical + 1
    elif msg_pt > hlc.physical:
        hlc.physical = msg_pt
        hlc.logical = msg_l + 1
    else:  // hlc.physical == msg_pt
        hlc.logical = max(hlc.logical, msg_l) + 1
```

### Properties

| Property | HLC | Lamport | Vector |
|----------|-----|---------|--------|
| Preserves happens-before | ✅ | ✅ | ✅ |
| Detects concurrency | ❌ | ❌ | ✅ |
| Close to physical time | ✅ | ❌ | ❌ |
| Constant size per event | ✅ | ✅ | ❌ (O(N)) |
| Useful for snapshots | ✅ | ❌ | ✅ |

HLCs are used by CockroachDB, MongoDB, and other databases that need causally consistent snapshots tied to approximate wall-clock time.

## Causal Ordering in Practice

### Causal Broadcast

A message delivery protocol is **causally ordered** if: whenever a message `m1` causally precedes `m2`, every process delivers `m1` before `m2`.

```
Implementation sketch (using vector clocks):

On broadcast(m) at process i:
    V[i] = V[i] + 1
    send (m, V) to all processes

On receive(m, T) at process j:
    buffer the message until:
        T[i] == V[i] + 1          (this is the next expected from sender i)
        T[k] ≤ V[k] for all k ≠ i (all causal predecessors delivered)
    then:
        deliver m
        V[i] = V[i] + 1
```

### Total Order Broadcast

A stronger guarantee: all processes deliver all messages in the **same order**. This is equivalent to consensus — implementing total order broadcast requires solving the consensus problem (Paxos, Raft, etc., covered in Section 4).

| Ordering guarantee | Strength | Requires consensus? |
|-------------------|----------|-------------------|
| FIFO (per-sender) | Weakest | No |
| Causal | Medium | No (vector clocks suffice) |
| Total | Strongest | Yes |

## Key Takeaways

- Physical clocks are unreliable for ordering events across nodes — clock skew, drift, and jumps make timestamp comparison unsafe
- The happens-before relation captures causality without physical time: if `a → b`, then `a` could have caused `b`
- Lamport clocks provide a consistent counter that respects happens-before, but cannot detect concurrent events
- Vector clocks capture full causal history and detect concurrency, but have O(N) space cost per event
- Hybrid Logical Clocks combine physical time with logical counters — constant size, close to wall-clock time, and used by CockroachDB and MongoDB
- Causal broadcast delivers messages respecting causal order without consensus; total order broadcast requires consensus
