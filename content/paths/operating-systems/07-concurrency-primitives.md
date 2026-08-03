---
title: "Concurrency Primitives"
weight: 7
---

# Concurrency Primitives

Concurrency is the art of managing multiple threads of execution that share resources. Without proper synchronization, programs exhibit **race conditions** — subtle, non-deterministic bugs that are notoriously difficult to reproduce and fix.

---

## Race Conditions and Critical Sections

A **race condition** occurs when the outcome depends on the timing of thread execution:

```python
counter = 0

def increment():
    global counter
    # NOT atomic — three operations:
    # 1. Read counter into register
    # 2. Add 1
    # 3. Write back to counter
    counter += 1

# Two threads running increment() 1000 times each
# Expected: 2000  |  Actual: unpredictable (e.g., 1847)
```

A **critical section** is a code region accessing shared resources that must not execute concurrently.

### Requirements for Correct Synchronization

| Property | Meaning |
|----------|---------|
| **Mutual exclusion** | At most one thread in the critical section at a time |
| **Progress** | If no thread is in the CS, a waiting thread can enter |
| **Bounded waiting** | No thread waits forever (no starvation) |

---

## Mutexes (Mutual Exclusion Locks)

The simplest synchronization primitive — a binary lock:

```python
import threading

counter = 0
lock = threading.Lock()

def safe_increment():
    global counter
    with lock:          # acquire on enter, release on exit
        counter += 1    # critical section
```

### Spinlock vs Sleeping Mutex

| Type | Behavior | Best For |
|------|----------|----------|
| **Spinlock** | Busy-wait in a loop | Very short critical sections |
| **Sleeping mutex** | Block thread, wake on release | Longer critical sections, I/O |

```python
# Pseudocode: Spinlock
class SpinLock:
    def __init__(self):
        self.locked = AtomicBool(False)

    def acquire(self):
        while self.locked.test_and_set():  # atomic CAS
            pass  # spin

    def release(self):
        self.locked.clear()
```

---

## Semaphores

A **semaphore** is a generalized lock with a counter:

```python
import threading

# Counting semaphore — allow N concurrent accesses
pool_sem = threading.Semaphore(5)

def use_resource():
    pool_sem.acquire()   # decrement; block if count == 0
    try:
        access_limited_resource()
    finally:
        pool_sem.release()  # increment; wake one waiting thread
```

### Semaphore Operations

| Operation | Effect |
|-----------|--------|
| `wait()` / `acquire()` / `P()` | Decrement counter; block if counter < 0 |
| `signal()` / `release()` / `V()` | Increment counter; wake one blocked thread |

### Signaling Between Threads

```python
event = threading.Semaphore(0)  # starts at 0

def thread_a():
    do_work()
    event.release()  # signal completion

def thread_b():
    event.acquire()  # block until thread_a signals
    proceed_after_a()
```

---

## Monitors and Condition Variables

A **monitor** bundles mutual exclusion with condition synchronization:

```python
import threading

class BoundedBuffer:
    def __init__(self, capacity):
        self.buffer = []
        self.capacity = capacity
        self.lock = threading.Lock()
        self.not_full = threading.Condition(self.lock)
        self.not_empty = threading.Condition(self.lock)

    def produce(self, item):
        with self.not_full:
            while len(self.buffer) >= self.capacity:
                self.not_full.wait()    # release lock + sleep
            self.buffer.append(item)
            self.not_empty.notify()     # wake one consumer

    def consume(self):
        with self.not_empty:
            while len(self.buffer) == 0:
                self.not_empty.wait()   # release lock + sleep
            item = self.buffer.pop(0)
            self.not_full.notify()      # wake one producer
            return item
```

> Always use `while` (not `if`) for condition checks — **spurious wakeups** can occur.

---

## Deadlock

**Deadlock** occurs when threads are permanently blocked, each waiting for a resource held by another.

### Four Necessary Conditions (Coffman Conditions)

| Condition | Meaning |
|-----------|---------|
| **Mutual exclusion** | Resources cannot be shared |
| **Hold and wait** | Thread holds resources while waiting for others |
| **No preemption** | Resources cannot be forcibly taken |
| **Circular wait** | Circular chain of dependencies |

### Example

```python
lock_a = threading.Lock()
lock_b = threading.Lock()

def thread_1():
    lock_a.acquire()    # holds A
    lock_b.acquire()    # waits for B → DEADLOCK

def thread_2():
    lock_b.acquire()    # holds B
    lock_a.acquire()    # waits for A → DEADLOCK
```

### Deadlock Prevention

| Strategy | Breaks | How |
|----------|--------|-----|
| **Lock ordering** | Circular wait | Always acquire locks in global order |
| **Lock timeout** | Hold and wait | Release all locks if timeout expires |
| **All-or-nothing** | Hold and wait | Acquire all locks atomically or none |

```python
# Prevention: consistent lock ordering
def thread_1():
    lock_a.acquire()  # always A first
    lock_b.acquire()

def thread_2():
    lock_a.acquire()  # always A first (same order!)
    lock_b.acquire()
```

### Deadlock Avoidance — Banker's Algorithm

The OS only grants requests that keep the system in a **safe state**:

```
Available: [3, 3, 2]
         Max    Allocated    Needs
P0:    [7,5,3]   [0,1,0]   [7,4,3]
P1:    [3,2,2]   [2,0,0]   [1,2,2]
P2:    [9,0,2]   [3,0,2]   [6,0,0]

Safe sequence: <P1, P0, P2> → grant request
No safe sequence → deny (would lead to potential deadlock)
```

### Deadlock Detection and Recovery

- **Detection:** Build resource-allocation graph; detect cycles periodically
- **Recovery:** Kill one process in the cycle, or preempt its resources

---

## Classic Problems

### Reader-Writer Problem

```python
class ReadWriteLock:
    def __init__(self):
        self.readers = 0
        self.lock = threading.Lock()
        self.write_lock = threading.Lock()

    def read_acquire(self):
        with self.lock:
            self.readers += 1
            if self.readers == 1:
                self.write_lock.acquire()  # first reader blocks writers

    def read_release(self):
        with self.lock:
            self.readers -= 1
            if self.readers == 0:
                self.write_lock.release()  # last reader unblocks writers

    def write_acquire(self):
        self.write_lock.acquire()

    def write_release(self):
        self.write_lock.release()
```

### Producer-Consumer with Semaphores

```python
from collections import deque

class ProducerConsumer:
    def __init__(self, maxsize):
        self.queue = deque()
        self.mutex = threading.Lock()
        self.items = threading.Semaphore(0)
        self.spaces = threading.Semaphore(maxsize)

    def produce(self, item):
        self.spaces.acquire()      # wait for empty slot
        with self.mutex:
            self.queue.append(item)
        self.items.release()       # signal new item

    def consume(self):
        self.items.acquire()       # wait for item
        with self.mutex:
            item = self.queue.popleft()
        self.spaces.release()      # signal slot freed
        return item
```

### Dining Philosophers

```python
forks = [threading.Lock() for _ in range(5)]

def philosopher(i):
    while True:
        think()
        first = min(i, (i + 1) % 5)   # prevent circular wait
        second = max(i, (i + 1) % 5)
        forks[first].acquire()
        forks[second].acquire()
        eat()
        forks[second].release()
        forks[first].release()
```

---

## Modern Concurrency Abstractions

| Abstraction | Level | Example |
|-------------|-------|---------|
| Atomic operations | Hardware | `AtomicInteger`, compare-and-swap |
| Lock-free structures | Library | Concurrent queues, skip lists |
| Channels | Language | Go channels, Rust mpsc |
| Actors | Framework | Erlang processes, Akka |
| async/await | Language | Python asyncio, Rust tokio |

---

## Key Takeaways

1. **Every shared mutable state** needs synchronization — no exceptions.
2. Use the **simplest primitive** that solves the problem: mutex > semaphore > complex schemes.
3. Always acquire locks in a **consistent global order** to prevent deadlock.
4. Use `while` loops when waiting on conditions — spurious wakeups are real.
5. **Prefer higher-level abstractions** (channels, actors, async/await) over raw locks.
6. Deadlock requires all four Coffman conditions — breaking any one prevents it.
7. The producer-consumer pattern is a foundational building block for message queues, thread pools, and pipelines.
