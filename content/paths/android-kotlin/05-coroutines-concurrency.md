---
title: "Coroutines and Concurrency"
weight: 5
---

# Coroutines and Concurrency

Kotlin coroutines provide a structured, lightweight model for asynchronous and concurrent programming. Unlike threads, coroutines are cheap to create (thousands can run simultaneously), and structured concurrency ensures that work is properly scoped and cancelled.

## Suspend Functions

A `suspend` function can be paused and resumed without blocking a thread. Suspension is the fundamental building block of coroutines:

```kotlin
suspend fun fetchUser(id: Long): User {
    // This call suspends the coroutine, freeing the thread
    val response = httpClient.get("https://api.example.com/users/$id")
    return response.body()
}

suspend fun fetchUserWithPosts(id: Long): UserWithPosts {
    val user = fetchUser(id)                    // Sequential
    val posts = fetchPosts(user.id)             // Sequential — waits for user first
    return UserWithPosts(user, posts)
}
```

Suspend functions can only be called from other suspend functions or from a coroutine builder.

### Concurrent Decomposition with async

When two operations are independent, use `async` to run them concurrently:

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val user = async { fetchUser(currentUserId) }
    val notifications = async { fetchNotifications(currentUserId) }
    val feed = async { fetchFeed() }

    // All three requests run concurrently
    Dashboard(
        user = user.await(),
        notifications = notifications.await(),
        feed = feed.await()
    )
}
```

## CoroutineScope and Builders

Every coroutine runs inside a `CoroutineScope`, which defines the lifecycle and context of the coroutine:

```kotlin
// launch — fire-and-forget (returns Job)
scope.launch {
    val data = fetchData()
    updateUi(data)
}

// async — returns a Deferred<T> (future with a result)
val deferred: Deferred<User> = scope.async {
    fetchUser(123)
}
val user: User = deferred.await()

// runBlocking — blocks the current thread (for main functions and tests only)
fun main() = runBlocking {
    val result = fetchData()
    println(result)
}

// coroutineScope — creates a child scope, suspends until all children complete
suspend fun processAll(ids: List<Long>) = coroutineScope {
    ids.forEach { id ->
        launch { process(id) }
    }
    // Returns only when ALL launched coroutines finish
}
```

| Builder | Returns | Blocks thread? | Use case |
|---------|---------|----------------|----------|
| `launch` | `Job` | No | Fire-and-forget work |
| `async` | `Deferred<T>` | No | Concurrent computation with result |
| `runBlocking` | `T` | Yes | Main function, tests |
| `coroutineScope` | `T` | No (suspends) | Structured child scope |

## Dispatchers

Dispatchers determine which thread or thread pool a coroutine runs on:

```kotlin
launch(Dispatchers.Main) {
    // Main/UI thread — update UI elements
    textView.text = "Loading..."
}

launch(Dispatchers.IO) {
    // Optimised for blocking I/O (network, disk, database)
    val data = database.query("SELECT * FROM users")
}

launch(Dispatchers.Default) {
    // CPU-intensive work (sorting, parsing, computation)
    val sorted = largeList.sortedBy { it.score }
}

// Switch dispatcher within a coroutine
suspend fun loadAndDisplay() {
    val data = withContext(Dispatchers.IO) {
        fetchFromNetwork()         // Runs on IO thread pool
    }
    // Back on the original dispatcher
    displayData(data)
}
```

| Dispatcher | Thread pool | Use case |
|-----------|-------------|----------|
| `Dispatchers.Main` | Android main thread | UI updates |
| `Dispatchers.IO` | Shared, elastic pool | Network, disk, database |
| `Dispatchers.Default` | Shared, fixed pool (= CPU cores) | CPU-bound computation |
| `Dispatchers.Unconfined` | Caller's thread (then resumes on any) | Testing, rare edge cases |

## Structured Concurrency

Structured concurrency guarantees that coroutines follow a parent–child hierarchy. When a parent scope is cancelled, all children are cancelled too:

```kotlin
class UserRepository(private val scope: CoroutineScope) {

    fun refreshUsers() {
        scope.launch {
            // If scope is cancelled, this coroutine and its children stop
            val users = fetchUsers()
            val enriched = coroutineScope {
                users.map { user ->
                    async { enrichUser(user) }
                }.awaitAll()
            }
            cache.store(enriched)
        }
    }
}
```

### Cancellation

Cancellation is cooperative. Suspend functions from `kotlinx.coroutines` (like `delay`, `yield`, `withContext`) check for cancellation automatically. CPU-bound loops must check manually:

```kotlin
suspend fun processItems(items: List<Item>) = coroutineScope {
    for (item in items) {
        ensureActive()  // Throws CancellationException if cancelled
        process(item)
    }
}

// Cancel a specific job
val job = launch { longRunningWork() }
job.cancel()       // Request cancellation
job.join()         // Wait for it to finish
// Or combined:
job.cancelAndJoin()
```

### Exception Handling

```kotlin
// SupervisorJob — failure of one child does not cancel siblings
val supervisor = SupervisorJob()
val scope = CoroutineScope(Dispatchers.IO + supervisor)

scope.launch {
    throw RuntimeException("Oops")  // This child fails
}
scope.launch {
    delay(1000)
    println("Still running")  // This child continues
}

// CoroutineExceptionHandler — global catch
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught: ${exception.message}")
}

scope.launch(handler) {
    throw RuntimeException("Handled")
}
```

## Flow (Cold Streams)

A `Flow` emits multiple values over time, computed lazily. It is cold — nothing happens until a terminal operator collects it:

```kotlin
fun numberFlow(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)    // Emit a value downstream
    }
}

// Collect the flow
numberFlow().collect { value ->
    println(value)  // 1, 2, 3, 4, 5
}
```

### Flow Operators

Flows support transformation operators similar to collections:

```kotlin
fun userUpdates(): Flow<User> = flow { /* ... */ }

userUpdates()
    .filter { it.isActive }
    .map { it.name }
    .distinctUntilChanged()
    .onEach { println("User: $it") }
    .catch { e -> println("Error: ${e.message}") }
    .collect { name -> updateUi(name) }
```

| Operator | Purpose |
|----------|---------|
| `map` | Transform each emitted value |
| `filter` | Emit only values matching a predicate |
| `flatMapLatest` | Switch to a new flow on each emission, cancelling the previous |
| `debounce` | Wait for emissions to settle before emitting |
| `distinctUntilChanged` | Skip consecutive duplicate values |
| `catch` | Handle upstream exceptions |
| `onEach` | Side effect on each value (logging, analytics) |
| `combine` | Combine latest values from multiple flows |

### flowOf and asFlow

```kotlin
// Create flows from fixed values
val flow1 = flowOf(1, 2, 3)
val flow2 = listOf("a", "b", "c").asFlow()

// Combine multiple flows
val combined = combine(flow1, flow2) { number, letter ->
    "$number$letter"
}  // Emits combined latest values
```

## StateFlow and SharedFlow

StateFlow and SharedFlow are hot streams — they emit regardless of whether anyone is collecting.

### StateFlow

StateFlow holds a single current value and emits updates to all collectors. It is the reactive replacement for `LiveData`:

```kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() {
        _count.value++   // or _count.update { it + 1 }
    }
}

// Collecting in a coroutine
viewModel.count.collect { count ->
    textView.text = "Count: $count"
}
```

| Property | StateFlow | SharedFlow |
|----------|-----------|------------|
| Initial value | Required | Optional (via replay) |
| Current value | `.value` accessible | No direct access (unless replay > 0) |
| Replay | Always 1 (latest value) | Configurable (0, 1, N) |
| Equality | Skips emission if value unchanged | Emits every value |
| Use case | UI state, observable properties | Events, one-shot signals |

### SharedFlow

SharedFlow is for events that should not be replayed or that need configurable replay:

```kotlin
class EventBus {
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()

    suspend fun emit(event: Event) {
        _events.emit(event)
    }
}

// No replay by default — late collectors miss past events
// Configure replay:
val replayFlow = MutableSharedFlow<Event>(replay = 1)
```

### Converting Flow to StateFlow

```kotlin
class SearchViewModel : ViewModel() {
    private val query = MutableStateFlow("")

    val results: StateFlow<List<Item>> = query
        .debounce(300)
        .flatMapLatest { q -> repository.search(q) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}
```

## Channels

Channels provide a way to transfer values between coroutines, similar to blocking queues:

```kotlin
val channel = Channel<Int>(capacity = Channel.BUFFERED)

// Producer
launch {
    for (i in 1..5) {
        channel.send(i)
    }
    channel.close()
}

// Consumer
launch {
    for (value in channel) {
        println(value)
    }
}
```

| Channel Type | Behaviour |
|-------------|-----------|
| `Channel.RENDEZVOUS` (default) | Sender suspends until receiver is ready |
| `Channel.BUFFERED` | Buffer of 64, sender suspends when full |
| `Channel.UNLIMITED` | Unlimited buffer, sender never suspends |
| `Channel.CONFLATED` | Keeps only the latest value |

In most Android code, prefer Flow over Channels. Channels are useful for fan-out (multiple consumers) or when you need explicit send/receive semantics.

## Key Takeaways

- Suspend functions are the foundation of coroutines — they pause without blocking threads
- Use `launch` for fire-and-forget work, `async`/`await` for concurrent computations with results
- Dispatchers control threading: `Main` for UI, `IO` for network/disk, `Default` for CPU-bound work
- Structured concurrency ties coroutine lifecycles to scopes — parent cancellation cascades to all children
- Flow is a cold stream for reactive data; StateFlow holds the latest value (replaces LiveData); SharedFlow handles events
- Prefer `stateIn` to convert cold Flows into StateFlows for UI consumption, and use `WhileSubscribed` to manage upstream lifecycle
