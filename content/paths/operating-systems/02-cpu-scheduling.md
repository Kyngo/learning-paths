---
title: "CPU Scheduling"
weight: 2
---

# CPU Scheduling

The **CPU scheduler** decides which ready process gets the CPU next. This decision directly impacts system responsiveness, throughput, and fairness. A good scheduler makes a fast machine feel fast; a bad one makes it feel sluggish regardless of hardware.

---

## Scheduling Criteria

| Criterion | Definition | Optimize |
|-----------|-----------|----------|
| **CPU utilization** | Percentage of time CPU is busy | Maximize |
| **Throughput** | Processes completed per time unit | Maximize |
| **Turnaround time** | Total time from submission to completion | Minimize |
| **Waiting time** | Time spent in the ready queue | Minimize |
| **Response time** | Time from submission to first response | Minimize |

These criteria often conflict. Maximizing throughput may hurt response time; optimizing for interactive responsiveness may reduce batch throughput.

---

## Scheduling Metrics

For a process that arrives at time `arrival`, starts executing at `first_run`, and completes at `completion`:

```
Turnaround time = completion - arrival
Waiting time    = turnaround_time - burst_time
Response time   = first_run - arrival
```

---

## Preemptive vs Non-Preemptive

| Type | Behavior | Used By |
|------|----------|---------|
| **Non-preemptive** | Process runs until it voluntarily yields or terminates | FCFS, non-preemptive SJF |
| **Preemptive** | Scheduler can interrupt a running process | Round Robin, preemptive SJF, Linux CFS |

Modern general-purpose OS kernels are always preemptive — no user process can monopolize the CPU.

---

## First Come, First Served (FCFS)

The simplest algorithm: processes execute in arrival order.

```
Process   Burst Time   Arrival
P1        24           0
P2        3            0
P3        3            0

Gantt chart:
|-------- P1 --------|-- P2 --|-- P3 --|
0                    24      27      30

Waiting times: P1=0, P2=24, P3=27
Average waiting time: (0+24+27)/3 = 17
```

### Problems with FCFS

- **Convoy effect** — short processes stuck behind a long one
- Average waiting time is highly variable depending on arrival order
- Unacceptable for interactive systems

---

## Shortest Job First (SJF)

Select the process with the shortest next CPU burst.

```
Process   Burst Time   Arrival
P1        6            0
P2        8            0
P3        7            0
P4        3            0

SJF order: P4(3) → P1(6) → P3(7) → P2(8)

Gantt chart:
|P4-|--- P1 ---|---- P3 ----|------ P2 ------|
0   3         9           16              24

Waiting times: P4=0, P1=3, P3=9, P2=16
Average waiting time: (0+3+9+16)/4 = 7
```

### SJF Properties

- **Optimal** — minimizes average waiting time (provably)
- **Impractical** — requires knowing future burst lengths
- **Approximation** — use exponential averaging of past bursts:

```
τ(n+1) = α × t(n) + (1 - α) × τ(n)

where:
  t(n)   = actual length of nth CPU burst
  τ(n)   = predicted length of nth burst
  α      = weight (typically 0.5)
```

### Preemptive SJF (Shortest Remaining Time First — SRTF)

If a new process arrives with a shorter remaining burst than the current process, preempt:

```
Process   Burst   Arrival
P1        8       0
P2        4       1
P3        9       2
P4        5       3

Timeline:
|P1|--- P2 ---|-- P4 --|---- P1 (remaining) ----|-------- P3 --------|
0  1         5        10                     17                    26
```

---

## Priority Scheduling

Each process gets a priority. The highest-priority ready process runs next.

| Priority type | Description |
|---------------|-------------|
| Static | Set at creation, never changes |
| Dynamic | Adjusted based on behavior (aging, I/O vs CPU) |

### Starvation Problem

Low-priority processes may never execute if high-priority ones keep arriving.

**Solution — Aging:** Gradually increase the priority of waiting processes:

```
effective_priority = base_priority + (wait_time / aging_factor)
```

---

## Round Robin (RR)

Each process gets a fixed **time quantum** (time slice). After the quantum expires, the process is preempted and placed at the end of the ready queue.

```
Time quantum = 4

Process   Burst
P1        24
P2        3
P3        3

|-- P1 --|P2-|P3-|-- P1 --|-- P1 --|-- P1 --|-- P1 --|-- P1 --|
0        4   7  10      14      18      22      26      30

Response times: P1=0, P2=4, P3=7
```

### Quantum Size Trade-offs

| Quantum | Effect |
|---------|--------|
| Too large | Degenerates to FCFS (poor response time) |
| Too small | Excessive context switching overhead |
| Typical | 10–100 ms (Linux default: ~4ms effective with CFS) |

**Rule of thumb:** 80% of CPU bursts should be shorter than the quantum.

---

## Multilevel Queue Scheduling

Processes are classified into groups, each with its own queue and scheduling algorithm:

```
┌─────────────────────────────────────────┐
│  System processes      (highest priority)│ ← FCFS
├─────────────────────────────────────────┤
│  Interactive processes                   │ ← Round Robin
├─────────────────────────────────────────┤
│  Batch processes       (lowest priority) │ ← FCFS
└─────────────────────────────────────────┘
```

### Multilevel Feedback Queue (MLFQ)

Processes can **move between queues** based on behavior:

```
Queue 0 (quantum=8ms):   ──▶ CPU burst > 8ms? Move to Queue 1
Queue 1 (quantum=16ms):  ──▶ CPU burst > 16ms? Move to Queue 2
Queue 2 (FCFS):          ──▶ Runs to completion (batch)
```

**Rules:**
1. Higher-priority queue always runs first
2. New processes start in the highest-priority queue
3. If a process uses its full quantum, it moves down
4. If a process blocks before quantum expires, it stays (or moves up)
5. Periodically boost all processes to top queue (prevent starvation)

---

## Real-Time Scheduling

Real-time systems have **deadlines** — missing a deadline is a failure.

| Type | Consequence of missed deadline | Example |
|------|-------------------------------|---------|
| **Hard real-time** | System failure / catastrophe | Pacemaker, ABS brakes |
| **Soft real-time** | Degraded quality (acceptable) | Video playback, VoIP |

### Rate Monotonic Scheduling (RMS)

- Static priorities based on period: shorter period = higher priority
- Utilization bound: `U ≤ n(2^(1/n) - 1)` ≈ 69% for large n

### Earliest Deadline First (EDF)

- Dynamic priority: process closest to its deadline runs next
- Theoretically optimal — can achieve 100% utilization
- Higher overhead than RMS

---

## Linux Completely Fair Scheduler (CFS)

Linux's default scheduler since kernel 2.6.23. It models an "ideal multitasking CPU" where every process runs simultaneously at 1/N speed.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Virtual runtime (vruntime)** | Track how much CPU time each process has received |
| **Red-black tree** | Ready processes sorted by vruntime |
| **Target latency** | Period in which all runnable processes should run once (~6ms) |
| **Minimum granularity** | Minimum time slice (~0.75ms) — prevents excessive switching |
| **Nice values** | -20 to +19 — affects vruntime accumulation rate |

### How CFS Works

```
1. Pick process with LOWEST vruntime from red-black tree  → O(1)
2. Run it for its calculated time slice
3. Update vruntime: vruntime += actual_runtime × (weight_nice0 / weight_process)
4. Reinsert into red-black tree                           → O(log n)
5. Repeat
```

A process with a higher nice value (lower priority) accumulates vruntime faster, so it gets scheduled less often.

### CFS Time Slice Calculation

```
time_slice = target_latency × (process_weight / total_weight_of_runnable)
```

If target_latency = 6ms and there are 3 equal-weight processes: each gets 2ms.

### CFS vs Traditional Schedulers

| Feature | O(1) Scheduler (old Linux) | CFS |
|---------|---------------------------|-----|
| Data structure | Bitmap + array | Red-black tree |
| Fairness | Approximate (heuristics) | Mathematically fair |
| Interactive detection | Complex heuristics | Emerges naturally (low vruntime) |
| Complexity | O(1) pick, complex logic | O(log n) pick, simple logic |

---

## Scheduling in Practice

### Multi-core Considerations

| Challenge | Solution |
|-----------|----------|
| Load balancing | Periodic migration of processes between CPUs |
| Cache affinity | Prefer running process on same CPU as last time |
| NUMA awareness | Schedule near process's memory allocation |
| Lock contention | Per-CPU run queues (Linux does this) |

### Linux Scheduling Classes (Priority Order)

```
1. SCHED_DEADLINE  — Earliest Deadline First (hard real-time)
2. SCHED_FIFO     — Real-time FIFO (no preemption within class)
3. SCHED_RR       — Real-time Round Robin
4. SCHED_NORMAL   — CFS (default for all user processes)
5. SCHED_BATCH    — CFS variant optimized for throughput
6. SCHED_IDLE     — Runs only when CPU is truly idle
```

---

## Comparison Summary

| Algorithm | Preemptive | Starvation | Optimal | Complexity | Best For |
|-----------|-----------|-----------|---------|------------|----------|
| FCFS | No | No | No | O(1) | Batch, simple systems |
| SJF | Optional | Yes | Yes (avg wait) | O(n) | Batch with known bursts |
| Priority | Optional | Yes (without aging) | No | O(n) | Systems with clear priorities |
| Round Robin | Yes | No | No | O(1) | Interactive, time-sharing |
| MLFQ | Yes | Possible | No | O(1) | General-purpose OS |
| CFS | Yes | No | Approximate | O(log n) | Linux general-purpose |

---

## Key Takeaways

- No single scheduling algorithm is optimal for all criteria simultaneously
- SJF minimizes average waiting time but requires future knowledge
- Round Robin provides fairness and bounded response time at the cost of throughput
- Multilevel feedback queues adapt to process behavior without prior knowledge
- Linux CFS achieves fairness through virtual runtime tracking in a red-black tree
- Real-time scheduling requires deadline guarantees, not just good average performance
- Modern schedulers must balance fairness, cache locality, and NUMA topology across many cores
