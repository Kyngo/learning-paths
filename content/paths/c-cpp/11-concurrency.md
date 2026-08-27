---
title: "Concurrency"
weight: 11
---

# Concurrency

C provides POSIX threads (`pthreads`). C++11 introduced a standard threading library with threads, mutexes, condition variables, futures, and atomics. C++20 added coroutines. This section covers both layers.

---

## C Threads (pthreads)

```c
#include <pthread.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d running\n", id);
    return NULL;
}

int main(void) {
    pthread_t threads[4];
    int ids[4] = {0, 1, 2, 3};

    for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }
    return 0;
}
```

```bash
gcc -pthread main.c -o app
```

### pthreads Mutex

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&lock);
// critical section
shared_counter++;
pthread_mutex_unlock(&lock);
```

---

## C++ Threads (`<thread>`)

```cpp
#include <thread>
#include <iostream>

void worker(int id) {
    std::cout << "Thread " << id << " running" << std::endl;
}

int main() {
    std::thread t1(worker, 1);
    std::thread t2(worker, 2);

    t1.join();   // wait for t1
    t2.join();   // wait for t2

    // Or detach (run independently — use with caution)
    // t1.detach();
}
```

### Lambda Threads

```cpp
int result = 0;
std::thread t([&result]() {
    result = expensive_computation();
});
t.join();
```

---

## Mutexes

```cpp
#include <mutex>

std::mutex mtx;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);  // RAII — unlocks on scope exit
    counter++;
}

// C++17 with CTAD
void increment() {
    std::lock_guard lock(mtx);  // type deduced
    counter++;
}

// unique_lock — more flexible (can unlock early, transfer ownership)
void process() {
    std::unique_lock lock(mtx);
    // ... critical section ...
    lock.unlock();  // release early
    // ... non-critical work ...
}
```

### Multiple Mutexes (Avoiding Deadlock)

```cpp
std::mutex m1, m2;

void safe_swap() {
    std::scoped_lock lock(m1, m2);  // C++17: locks both, deadlock-free
    // swap protected data
}
```

---

## Atomics

Lock-free operations on simple types:

```cpp
#include <atomic>

std::atomic<int> counter{0};

void increment() {
    counter++;                    // atomic increment
    counter.fetch_add(1);        // equivalent
    counter.store(42);           // atomic store
    int val = counter.load();    // atomic load
}

// Compare-and-swap (CAS)
int expected = 0;
counter.compare_exchange_strong(expected, 1);
// If counter == expected (0), set to 1 and return true
// Otherwise, set expected = current value and return false
```

### When to Use Atomics vs Mutexes

| Atomics | Mutexes |
|---------|---------|
| Single variable operations | Multi-variable operations |
| Counters, flags, pointers | Complex data structure updates |
| Lock-free, no blocking | Blocking (contention possible) |
| Simple types only | Any type |

---

## Condition Variables

Threads wait for a condition to become true:

```cpp
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;

// Producer
void produce(int value) {
    {
        std::lock_guard lock(mtx);
        queue.push(value);
    }
    cv.notify_one();  // wake one waiting consumer
}

// Consumer
void consume() {
    std::unique_lock lock(mtx);
    cv.wait(lock, [&] { return !queue.empty(); });  // wait until queue non-empty
    int value = queue.front();
    queue.pop();
    // process value
}
```

**Always use a predicate with `wait`** — spurious wakeups can occur.

---

## Async and Futures

```cpp
#include <future>

// Launch async task
auto future = std::async(std::launch::async, []() {
    return expensive_computation();
});

// Do other work...

int result = future.get();  // blocks until result is ready

// std::promise — manual future completion
std::promise<int> promise;
auto future = promise.get_future();

std::thread t([&promise]() {
    int result = compute();
    promise.set_value(result);
});

int value = future.get();
t.join();
```

---

## Thread Safety Patterns

### Thread-Safe Singleton (C++11 Magic Statics)

```cpp
Database& get_db() {
    static Database instance;  // initialised exactly once, thread-safe (C++11)
    return instance;
}
```

### Producer-Consumer Queue

```cpp
template<typename T>
class ThreadSafeQueue {
public:
    void push(T value) {
        std::lock_guard lock(mutex_);
        queue_.push(std::move(value));
        cv_.notify_one();
    }

    T pop() {
        std::unique_lock lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty(); });
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    bool try_pop(T& value) {
        std::lock_guard lock(mutex_);
        if (queue_.empty()) return false;
        value = std::move(queue_.front());
        queue_.pop();
        return true;
    }

private:
    std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
};
```

---

## Key Takeaways

- `std::lock_guard` and `std::scoped_lock` are the default mutex wrappers — RAII prevents forgotten unlocks.
- `std::atomic` is for simple single-variable operations. Mutexes are for everything else.
- Always use a predicate with condition variable `wait()` to handle spurious wakeups.
- `std::async` is the simplest way to run code asynchronously and get a result back.
- C++11 guarantees thread-safe initialisation of local statics — use this for singletons.
- `std::scoped_lock` (C++17) locks multiple mutexes simultaneously without deadlock.
