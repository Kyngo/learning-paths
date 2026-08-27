---
title: "Concurrency"
weight: 7
---

# Concurrency

Go was designed with concurrency as a first-class feature. Goroutines are lightweight threads managed by the Go runtime, and channels are typed conduits for communication between them. The motto: **"Don't communicate by sharing memory; share memory by communicating."**

---

## Goroutines

A goroutine is a lightweight concurrent function invocation. Starting one costs ~2 KB of stack (grows as needed) and is scheduled by the Go runtime, not the OS.

```go
go doWork()  // starts doWork as a goroutine

go func() {
    // anonymous goroutine
    fmt.Println("running concurrently")
}()
```

### Goroutines vs Threads

| | Goroutine | OS Thread |
|-|-----------|-----------|
| Stack size | ~2 KB (grows dynamically) | ~1 MB (fixed) |
| Creation cost | ~1 μs | ~100 μs |
| Scheduling | Go runtime (M:N) | OS kernel |
| Practical count | Millions | Thousands |
| Communication | Channels (safe by design) | Shared memory + locks |

---

## Channels

Channels are typed conduits for sending and receiving values between goroutines:

```go
ch := make(chan int)  // unbuffered channel of int

// Send
go func() {
    ch <- 42  // blocks until someone receives
}()

// Receive
value := <-ch  // blocks until someone sends
fmt.Println(value)  // 42
```

### Buffered Channels

```go
ch := make(chan int, 3)  // buffer size 3

ch <- 1  // does not block
ch <- 2  // does not block
ch <- 3  // does not block
ch <- 4  // BLOCKS — buffer full, waits for a receive
```

| Channel Type | Behaviour |
|-------------|-----------|
| Unbuffered `make(chan T)` | Send blocks until receive, and vice versa — synchronisation point |
| Buffered `make(chan T, n)` | Send blocks only when buffer is full; receive blocks when empty |

### Directional Channels

Restrict a channel to send-only or receive-only in function signatures:

```go
func producer(out chan<- int) {  // send-only
    for i := 0; i < 10; i++ {
        out <- i
    }
    close(out)
}

func consumer(in <-chan int) {  // receive-only
    for v := range in {
        fmt.Println(v)
    }
}
```

### Closing Channels

```go
close(ch)

// Receiving from a closed channel returns the zero value immediately
v := <-ch  // 0 for int (and ok=false)

// Range loop exits when channel is closed
for v := range ch {
    process(v)
}

// Check if channel is closed
v, ok := <-ch
if !ok {
    fmt.Println("channel closed")
}
```

**Rules:**
- Only the sender should close a channel
- Sending on a closed channel panics
- Receiving from a closed channel returns zero values

---

## Select

`select` is like `switch` for channels — it waits on multiple channel operations:

```go
select {
case msg := <-ch1:
    fmt.Println("received from ch1:", msg)
case msg := <-ch2:
    fmt.Println("received from ch2:", msg)
case ch3 <- value:
    fmt.Println("sent to ch3")
default:
    fmt.Println("no channel ready")  // non-blocking if included
}
```

### Timeout Pattern

```go
select {
case result := <-ch:
    fmt.Println(result)
case <-time.After(5 * time.Second):
    fmt.Println("timed out")
}
```

### Context Cancellation

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

select {
case result := <-doWork(ctx):
    fmt.Println(result)
case <-ctx.Done():
    fmt.Println("cancelled:", ctx.Err())
}
```

---

## Sync Primitives

### WaitGroup

Wait for a collection of goroutines to finish:

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        fmt.Println("worker", n)
    }(i)
}

wg.Wait()  // blocks until all Done() calls
```

### Mutex

Protect shared state:

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.v[key]++
}

func (c *SafeCounter) Value(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.v[key]
}
```

### RWMutex

Multiple readers, single writer:

```go
var mu sync.RWMutex

// Read lock — multiple goroutines can hold simultaneously
mu.RLock()
defer mu.RUnlock()

// Write lock — exclusive
mu.Lock()
defer mu.Unlock()
```

### Once

Execute something exactly once, regardless of how many goroutines call it:

```go
var once sync.Once
var db *sql.DB

func getDB() *sql.DB {
    once.Do(func() {
        db, _ = sql.Open("postgres", connStr)
    })
    return db
}
```

---

## Concurrency Patterns

### Fan-Out / Fan-In

```go
// Fan-out: distribute work across multiple goroutines
func fanOut(input <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        channels[i] = worker(input)
    }
    return channels
}

// Fan-in: merge multiple channels into one
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

### Pipeline

```go
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Usage: generate → square → consume
for v := range square(generate(1, 2, 3, 4)) {
    fmt.Println(v)  // 1, 4, 9, 16
}
```

### Worker Pool

```go
func workerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    go func() {
        wg.Wait()
        close(results)
    }()
}
```

### Graceful Shutdown

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    server := &http.Server{Addr: ":8080"}

    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    <-ctx.Done()  // wait for interrupt
    log.Println("shutting down...")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := server.Shutdown(shutdownCtx); err != nil {
        log.Fatal(err)
    }
    log.Println("server stopped")
}
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Goroutine leak | Goroutine blocks on channel forever | Use context cancellation or done channels |
| Race condition | Concurrent map/slice access | Use mutex or channels |
| Forgetting `wg.Add` before `go` | WaitGroup count wrong | Always `Add` before launching goroutine |
| Closing channel from receiver | Panic | Only senders close channels |
| No synchronisation at exit | `main` returns before goroutines finish | Use WaitGroup or channel receive |

### Race Detector

```bash
go run -race main.go
go test -race ./...
```

The race detector instruments memory access at runtime. **Always run tests with `-race`.**

---

## Key Takeaways

- Goroutines are cheap (~2 KB, ~1 μs to start). Use them freely for concurrent work.
- Channels synchronise goroutines. Unbuffered channels are synchronisation points; buffered channels are queues.
- `select` multiplexes channel operations. Combine with `context` for timeouts and cancellation.
- Use `sync.WaitGroup` to wait for goroutine completion, `sync.Mutex` for shared state, `sync.Once` for one-time initialisation.
- Always run with `-race` during development and testing. Data races are undefined behaviour.
- Prefer channels for orchestration, mutexes for protecting shared data structures.
