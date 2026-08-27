---
title: "Concurrency"
weight: 9
---

# Concurrency

Rust makes data races impossible at compile time. The type system (via `Send` and `Sync` traits) ensures that data shared between threads is always properly protected. If it compiles, there are no data races.

---

## Threads

```rust
use std::thread;

let handle = thread::spawn(|| {
    println!("hello from thread");
    42  // return value
});

let result = handle.join().unwrap();  // 42
```

### Moving Data into Threads

```rust
let name = String::from("Alice");

// ERROR: closure might outlive name
// thread::spawn(|| println!("{name}"));

// OK: move ownership into the thread
thread::spawn(move || {
    println!("{name}");
});
// name is no longer accessible here
```

### Scoped Threads (Rust 1.63+)

Scoped threads can borrow from the parent stack:

```rust
let mut data = vec![1, 2, 3];

thread::scope(|s| {
    s.spawn(|| {
        println!("{:?}", data);  // borrows data — OK because scope guarantees thread finishes
    });
    s.spawn(|| {
        // Can't mutably borrow data here — another thread is reading it
    });
});
// All scoped threads are joined before this point
```

---

## Send and Sync

Two marker traits that the compiler auto-implements:

| Trait | Meaning | Example |
|-------|---------|---------|
| `Send` | Safe to transfer ownership to another thread | Most types |
| `Sync` | Safe to share a reference (`&T`) between threads | Most types |
| `!Send` | Not safe to send | `Rc<T>`, raw pointers |
| `!Sync` | Not safe to share | `Cell<T>`, `RefCell<T>` |

```rust
// Arc<Mutex<T>> is Send + Sync — safe to share across threads
// Rc<RefCell<T>> is neither — single-thread only
```

The compiler checks these automatically. If you try to send a `!Send` type to another thread, the code will not compile.

---

## Shared State

### Mutex<T> — Mutual Exclusion

```rust
use std::sync::Mutex;

let counter = Mutex::new(0);

{
    let mut num = counter.lock().unwrap();  // acquire lock
    *num += 1;
}  // lock released when MutexGuard is dropped

println!("{}", counter.lock().unwrap());  // 1
```

### Arc<T> — Atomic Reference Counting

`Rc<T>` is single-threaded. `Arc<T>` (Atomic Reference Counted) is the thread-safe version:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    handles.push(thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    }));
}

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", counter.lock().unwrap());  // 10
```

`Arc<Mutex<T>>` is the standard pattern for shared mutable state across threads.

---

## Channels

Message passing — the alternative to shared state:

```rust
use std::sync::mpsc;  // multi-producer, single-consumer

let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send("hello").unwrap();
    tx.send("world").unwrap();
});

for msg in rx {
    println!("{msg}");
}
```

### Multiple Producers

```rust
let (tx, rx) = mpsc::channel();

for i in 0..5 {
    let tx = tx.clone();
    thread::spawn(move || {
        tx.send(i).unwrap();
    });
}
drop(tx);  // drop original sender so rx iterator terminates

let results: Vec<_> = rx.iter().collect();
```

---

## Async/Await

Rust's async model is based on **futures** — lazy values that represent a computation that will complete in the future.

```rust
async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(body)
}
```

### Runtimes

Rust's standard library does not include an async runtime. You choose one:

| Runtime | Use Case |
|---------|----------|
| `tokio` | Most popular, full-featured (networking, file I/O, timers) |
| `async-std` | std-like API |
| `smol` | Minimal, lightweight |

### Tokio Example

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get("https://httpbin.org/get").await?;
    println!("Status: {}", response.status());
    Ok(())
}
```

### Concurrency with Async

```rust
use tokio::task;

// Spawn concurrent tasks
let handle1 = task::spawn(async { fetch_data("url1").await });
let handle2 = task::spawn(async { fetch_data("url2").await });

let (r1, r2) = tokio::join!(handle1, handle2);

// Select — first to complete wins
tokio::select! {
    val = future1 => println!("future1 completed: {val:?}"),
    val = future2 => println!("future2 completed: {val:?}"),
}
```

### Async vs Threads

| | `std::thread` | `tokio` async |
|-|---------------|--------------|
| Overhead | ~8 KB stack per thread | ~few hundred bytes per task |
| Count | Thousands | Millions |
| I/O model | Blocking | Non-blocking |
| CPU-bound work | Good | Use `spawn_blocking` |
| I/O-bound work | Wastes threads waiting | Efficient multiplexing |

---

## Key Takeaways

- `Send` and `Sync` make data races a compile error. If it compiles, it is thread-safe.
- `Arc<Mutex<T>>` is the standard shared-state pattern. `Arc` for sharing, `Mutex` for mutation.
- Channels (`mpsc`) provide message-passing concurrency — no locks needed.
- Async/await is for I/O-bound concurrency. Rust requires an external runtime (tokio is the standard).
- Use `std::thread` for CPU-bound parallelism, `tokio` tasks for I/O-bound concurrency.
- Scoped threads (1.63+) allow borrowing from the parent scope — often simpler than `Arc`.
