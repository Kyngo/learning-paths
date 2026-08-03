---
title: "Processes & Threads"
weight: 1
---

# Processes & Threads

A **process** is the OS's abstraction for a running program. It encapsulates the program's code, data, and execution state into an isolated unit that the OS can manage, schedule, and protect.

---

## The Process Model

A process is more than just executing code. It includes:

| Component | Description |
|-----------|-------------|
| Text segment | The compiled program instructions |
| Data segment | Global and static variables |
| Heap | Dynamically allocated memory (grows upward) |
| Stack | Function call frames, local variables (grows downward) |
| Registers | CPU register state (PC, SP, general-purpose) |
| Open files | File descriptors the process holds |
| Credentials | User ID, group ID, permissions |

### Process Memory Layout

```
High addresses
┌──────────────────┐
│      Stack       │ ← grows downward
│        ↓         │
├──────────────────┤
│                  │
│   (free space)   │
│                  │
├──────────────────┤
│        ↑         │
│       Heap       │ ← grows upward
├──────────────────┤
│   BSS (uninit)   │
├──────────────────┤
│   Data (init)    │
├──────────────────┤
│      Text        │ ← program code
└──────────────────┘
Low addresses
```

---

## Process States

Every process transitions through a well-defined set of states:

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
┌─────┐  admit  ┌───────┐  dispatch  ┌─────────┐   terminate   ┌────────────┐
│ New │────────▶│ Ready │──────────▶│ Running │──────────────▶│ Terminated │
└─────┘         └───────┘           └─────────┘              └────────────┘
                    ▲                     │
                    │                     │ I/O or event wait
                    │    I/O or event     │
                    │    completion       ▼
                    │               ┌─────────┐
                    └───────────────│ Waiting │
                                   └─────────┘
```

| State | Description |
|-------|-------------|
| **New** | Process is being created |
| **Ready** | Waiting to be assigned to a CPU |
| **Running** | Instructions are being executed |
| **Waiting** | Waiting for I/O or an event |
| **Terminated** | Finished execution, awaiting cleanup |

### Additional States in Real Systems

- **Zombie** — process has terminated but parent hasn't called `wait()` yet
- **Orphan** — parent terminated before child; adopted by init/systemd
- **Stopped** — halted by a signal (e.g., SIGSTOP), can be resumed

---

## Process Control Block (PCB)

The OS maintains a **PCB** (also called task_struct in Linux) for every process:

```c
struct pcb {
    int pid;                    // Process ID
    int state;                  // Current state
    int priority;               // Scheduling priority
    unsigned long pc;           // Program counter
    unsigned long sp;           // Stack pointer
    unsigned long registers[N]; // Saved CPU registers
    struct mm_struct *memory;   // Memory management info
    struct files_struct *files; // Open file descriptors
    struct pcb *parent;         // Parent process
    struct list_head children;  // Child processes
    unsigned long cpu_time;     // Accumulated CPU time
    int exit_code;              // Exit status
};
```

The PCB is the kernel's complete snapshot of a process. During a context switch, the current process's state is saved into its PCB, and the next process's state is loaded from its PCB.

---

## Context Switching

A **context switch** occurs when the CPU switches from one process to another:

1. Save the current process's registers into its PCB
2. Update the process state (running → ready/waiting)
3. Select the next process (via scheduler)
4. Load the new process's registers from its PCB
5. Flush/update the TLB (translation lookaside buffer)
6. Resume execution at the new process's program counter

### Cost of Context Switching

| Factor | Impact |
|--------|--------|
| Register save/restore | ~microseconds (direct CPU cost) |
| TLB flush | Cache misses on memory access after switch |
| Cache pollution | L1/L2/L3 caches now hold stale data |
| Pipeline flush | CPU pipeline must be drained |
| Scheduler overhead | Decision time to pick next process |

**Typical cost:** 1–10 microseconds on modern hardware, but the indirect costs (cache/TLB misses) can multiply this by 10–100x in terms of execution slowdown.

---

## Threads vs Processes

A **thread** is a lightweight unit of execution within a process. Multiple threads share the same address space.

| Aspect | Process | Thread |
|--------|---------|--------|
| Address space | Own isolated space | Shared with other threads |
| Creation cost | Expensive (fork + copy page tables) | Cheap (share existing space) |
| Context switch | Expensive (TLB flush) | Cheaper (same address space) |
| Communication | IPC required (pipes, sockets, shared memory) | Direct shared memory access |
| Isolation | Full isolation (fault in one ≠ crash another) | No isolation (one thread crash = process crash) |
| Overhead | Higher (separate page tables, file descriptors) | Lower (shared resources) |

### What Threads Share vs Own

```
Shared across threads:          Private to each thread:
├── Code (text segment)         ├── Thread ID
├── Global data                 ├── Program counter
├── Heap                        ├── Register set
├── Open files                  ├── Stack
├── Signal handlers             └── Thread-local storage (TLS)
├── Process ID
└── Address space
```

---

## User-Level vs Kernel-Level Threads

| Aspect | User-Level Threads | Kernel-Level Threads |
|--------|-------------------|---------------------|
| Managed by | User-space library (green threads) | OS kernel |
| Scheduling | Library scheduler | OS scheduler |
| Context switch | Fast (no kernel involvement) | Slower (system call required) |
| Blocking I/O | Blocks entire process | Only blocks that thread |
| Parallelism | No true parallelism (single kernel thread) | True multi-core parallelism |
| Examples | Go goroutines (M:N), Java green threads (historical) | pthreads, Windows threads |

### Threading Models

```
1:1 (One-to-One)           M:1 (Many-to-One)        M:N (Many-to-Many)
┌───┐ ┌───┐ ┌───┐         ┌───┐┌───┐┌───┐          ┌───┐┌───┐┌───┐
│UT1│ │UT2│ │UT3│         │UT1││UT2││UT3│          │UT1││UT2││UT3│
└─┬─┘ └─┬─┘ └─┬─┘         └─┬─┘└─┬─┘└─┬─┘          └─┬─┘└─┬─┘└─┬─┘
  │     │     │              └──┬──┘──┘                └──┬──┘  │
  ▼     ▼     ▼                 ▼                         ▼     ▼
┌───┐ ┌───┐ ┌───┐           ┌───┐                     ┌───┐ ┌───┐
│KT1│ │KT2│ │KT3│           │KT1│                     │KT1│ │KT2│
└───┘ └───┘ └───┘           └───┘                     └───┘ └───┘

Linux pthreads              Older green threads        Go runtime, Erlang
```

---

## Thread Pools

Creating a thread per request is expensive. **Thread pools** pre-create a fixed set of worker threads:

```
┌─────────────────────────────────────────────┐
│                Thread Pool                   │
│  ┌────────┐ ┌────────┐ ┌────────┐          │
│  │Worker 1│ │Worker 2│ │Worker 3│  ...      │
│  └────┬───┘ └────┬───┘ └────┬───┘          │
│       │          │          │               │
│       └──────────┴──────────┘               │
│                  ▲                          │
│            ┌─────┴─────┐                    │
│            │ Task Queue │                    │
│            └───────────┘                    │
└─────────────────────────────────────────────┘
                   ▲
          Submit tasks here
```

### Advantages of Thread Pools

- **Bounded resource usage** — limit maximum concurrency
- **Amortized creation cost** — threads are reused, not created/destroyed per task
- **Backpressure** — queue depth signals overload
- **Graceful degradation** — excess work queues instead of spawning unbounded threads

### Sizing a Thread Pool

| Workload type | Optimal pool size |
|---------------|-------------------|
| CPU-bound | Number of CPU cores |
| I/O-bound | Cores × (1 + wait_time / compute_time) |
| Mixed | Profile and tune empirically |

---

## Process Creation

### fork() + exec() (Unix/Linux)

```c
pid_t pid = fork();  // Create child (copy of parent)

if (pid == 0) {
    // Child process
    exec("/bin/ls", args);  // Replace with new program
} else {
    // Parent process
    wait(&status);  // Wait for child to finish
}
```

- `fork()` creates a near-identical copy (with copy-on-write optimization)
- `exec()` replaces the process image with a new program
- This two-step model allows the child to set up redirections between fork and exec

### Modern Alternatives

| Call | Description |
|------|-------------|
| `posix_spawn()` | Combined fork+exec (avoids copy overhead) |
| `clone()` | Linux-specific, fine-grained sharing control |
| `vfork()` | Shares parent's address space until exec (dangerous, mostly deprecated) |

---

## Key Takeaways

- A process is the unit of resource ownership and protection; a thread is the unit of execution
- Context switches are expensive primarily due to cache/TLB invalidation, not register saves
- The 1:1 threading model (one user thread = one kernel thread) dominates modern systems
- Thread pools bound concurrency and amortize thread creation cost
- Linux represents both processes and threads as `task_struct` — threads are processes that share an address space
- Zombie processes waste PID table entries; always `wait()` on children
