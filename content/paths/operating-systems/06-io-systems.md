---
title: "I/O Systems"
weight: 6
---

# I/O Systems

Input/Output systems bridge the gap between the CPU and the external world — disks, networks, keyboards, GPUs. Understanding how the OS manages I/O is critical for writing performant software and diagnosing bottlenecks.

---

## I/O Hardware Fundamentals

Every I/O device connects to the system through a **controller** that exposes registers to the CPU:

| Register | Purpose |
|----------|---------|
| **Status** | Indicates device state (busy, ready, error) |
| **Command** | CPU writes operations (read, write, seek) |
| **Data-in** | CPU reads data from device |
| **Data-out** | CPU writes data to device |

### Port-Mapped vs Memory-Mapped I/O

| Approach | Mechanism | Example |
|----------|-----------|---------|
| Port-mapped | Special CPU instructions (`in`/`out`) access device registers | x86 legacy devices |
| Memory-mapped | Device registers mapped to memory addresses | Modern GPUs, NVMe |

Memory-mapped I/O is dominant today — it allows the same load/store instructions to access both RAM and devices.

---

## Data Transfer Mechanisms

### Polling (Programmed I/O)

The CPU repeatedly checks the device status register in a tight loop:

```c
while (device->status != READY) {
    // spin — wastes CPU cycles
}
device->data_out = buffer[i];
device->command = WRITE;
```

**Pros:** Simple, low latency for fast devices.
**Cons:** Wastes CPU cycles; unsuitable for slow devices.

### Interrupts

The device signals the CPU when ready, freeing the CPU to do other work:

```
1. CPU initiates I/O, continues other work
2. Device completes operation → raises interrupt
3. CPU saves context → jumps to ISR (Interrupt Service Routine)
4. ISR processes data → returns control
```

**Pros:** CPU is free between operations.
**Cons:** Context switch overhead; interrupt storms under high load.

### Direct Memory Access (DMA)

A dedicated DMA controller transfers data between device and memory without CPU involvement:

```
1. CPU programs DMA: source, destination, byte count
2. DMA controller transfers data directly to/from RAM
3. DMA raises interrupt when transfer completes
4. CPU only involved at start and end
```

### Comparison

| Method | CPU Usage | Latency | Best For |
|--------|-----------|---------|----------|
| Polling | 100% during transfer | Lowest | Fast devices, short transfers |
| Interrupts | Low (only during ISR) | Medium | General purpose |
| DMA | Minimal | Higher setup cost | Bulk transfers (disk, network) |

---

## I/O Scheduling

For rotational disks, the order of requests matters because seek time dominates performance.

### Elevator Algorithms

| Algorithm | Strategy | Pros | Cons |
|-----------|----------|------|------|
| **FCFS** | First come, first served | Fair | Random seeks, poor throughput |
| **SSTF** | Shortest Seek Time First | Good throughput | Starvation of distant requests |
| **SCAN** (Elevator) | Move in one direction, reverse at end | No starvation | End-of-disk latency |
| **C-SCAN** | One direction only, jump back to start | Uniform wait time | Slightly lower throughput |
| **LOOK / C-LOOK** | Like SCAN/C-SCAN but reverse at last request | Practical optimization | Standard in Linux |

```
Disk positions: 0 ----[20]----[45]----[67]----[89]---- 199
Head at 50, moving right:

SCAN:   50 → 67 → 89 → 199 → 45 → 20
LOOK:   50 → 67 → 89 → 45 → 20  (no travel to 199)
C-LOOK: 50 → 67 → 89 → 20 → 45  (jump back, serve in one direction)
```

> **Note:** SSDs have no seek penalty, so I/O scheduling focuses on **queue depth** and **parallelism** (Linux `mq-deadline` or `none`).

---

## Blocking vs Non-Blocking I/O

| Model | Behavior | Use Case |
|-------|----------|----------|
| **Blocking** | Thread sleeps until I/O completes | Simple programs, one connection per thread |
| **Non-blocking** | Returns immediately (EAGAIN if not ready) | Polling loops, game engines |
| **I/O Multiplexing** | Monitor multiple FDs, block until any ready | Servers (select, poll, epoll) |
| **Async I/O** | Kernel completes I/O and notifies | High-performance servers (io_uring) |

### The C10K Problem

```
Thread-per-connection:  10,000 threads x 8MB stack = 80GB RAM  ✗
Event-driven (epoll):   1 thread monitoring 10,000 FDs         ✓
```

---

## Modern I/O Multiplexing

### epoll (Linux)

```c
int epfd = epoll_create1(0);
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);  // register once

// O(1) per ready event — only returns active FDs
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout);
for (int i = 0; i < n; i++) {
    handle(events[i]);
}
```

### kqueue (BSD/macOS)

```c
int kq = kqueue();
EV_SET(&change, fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
int n = kevent(kq, &change, 1, events, MAX_EVENTS, &timeout);
```

### io_uring (Linux 5.1+)

```c
struct io_uring ring;
io_uring_queue_init(32, &ring, 0);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, len, offset);
io_uring_submit(&ring);

struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
```

| Mechanism | Platforms | Scalability | Syscalls per op |
|-----------|-----------|-------------|-----------------|
| select/poll | All POSIX | O(n) | 1 per wait |
| epoll | Linux | O(1) | 1 per wait |
| kqueue | BSD/macOS | O(1) | 1 per wait |
| io_uring | Linux 5.1+ | O(1) | 0 (shared ring) |

---

## Buffering Strategies

| Strategy | Description | Trade-off |
|----------|-------------|-----------|
| **No buffering** | Transfer directly to user space | Minimum latency, maximum syscalls |
| **Single buffer** | One kernel buffer per transfer | Balances latency and throughput |
| **Double buffer** | Fill one while processing other | Higher throughput, more memory |
| **Circular buffer** | Ring of buffers (producer/consumer) | Best for streaming, bounded memory |

### Page Cache

Linux uses free RAM as a **page cache** for disk I/O:
- **Read:** Check cache first; hit = no disk I/O
- **Write:** Write to cache (dirty page), flush asynchronously
- `O_DIRECT` bypasses the cache for databases that manage their own

---

## Disk vs SSD Characteristics

| Property | HDD | SSD (NAND) | NVMe SSD |
|----------|-----|------------|----------|
| Random read latency | 5-15 ms | 50-100 us | 10-20 us |
| Sequential throughput | 100-200 MB/s | 500-600 MB/s | 3-7 GB/s |
| IOPS (random 4K) | 100-200 | 50K-100K | 500K-1M |
| Wear mechanism | Mechanical fatigue | Write amplification | Same as SSD |
| I/O scheduler | Elevator essential | `mq-deadline` or `none` | `none` |
| Queue depth | 1 (NCQ: 32) | 32 | 65,535 per queue |

### Why SSDs Change Everything

- No seek penalty means random reads approximate sequential reads
- Internal parallelism benefits from high queue depth
- Write amplification means the OS should batch small writes
- TRIM informs SSD of freed blocks for garbage collection

---

## Key Takeaways

1. **DMA** is the standard for bulk transfers — polling and interrupts are for low-bandwidth or latency-critical paths.
2. **I/O scheduling** matters for HDDs (elevator algorithms) but is mostly irrelevant for SSDs.
3. **epoll/kqueue** solved the C10K problem; **io_uring** is solving C10M with zero-syscall async I/O.
4. The **page cache** makes repeated reads free and batches writes — understand when to bypass it.
5. SSDs have fundamentally different characteristics — design for parallelism and queue depth, not sequential access.
6. Choose the right I/O model: blocking for simplicity, event-driven for scale, io_uring for maximum throughput.
