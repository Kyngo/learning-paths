---
title: "Collections and Sequences"
weight: 3
---

# Collections and Sequences

Kotlin's collections API is one of the language's strongest features. It provides a rich set of operations for transforming, filtering, and aggregating data, while making a clear distinction between read-only and mutable collections. Sequences extend this model with lazy evaluation for processing large datasets efficiently.

## Collection Hierarchy

Kotlin separates read-only interfaces from mutable ones at the type level:

| Read-Only | Mutable | Factory function |
|-----------|---------|-----------------|
| `List<T>` | `MutableList<T>` | `listOf()` / `mutableListOf()` |
| `Set<T>` | `MutableSet<T>` | `setOf()` / `mutableSetOf()` |
| `Map<K, V>` | `MutableMap<K, V>` | `mapOf()` / `mutableMapOf()` |

Read-only collections have no `add`, `remove`, or `put` methods. This is enforced at the interface level, not by creating defensive copies:

```kotlin
val names: List<String> = listOf("Alice", "Bob", "Charlie")
// names.add("Dave")  // Compile error — List has no add()

val mutableNames: MutableList<String> = mutableListOf("Alice", "Bob")
mutableNames.add("Charlie")  // OK
```

### List

Ordered collection that allows duplicates:

```kotlin
val fruits = listOf("apple", "banana", "cherry", "apple")
fruits[0]              // "apple"
fruits.size            // 4
fruits.contains("banana")  // true
fruits.indexOf("cherry")   // 2

// Empty list with explicit type
val empty = emptyList<String>()

// Build a list programmatically
val squares = List(5) { it * it }  // [0, 1, 4, 9, 16]
```

### Set

Unordered collection with no duplicates:

```kotlin
val uniqueNames = setOf("Alice", "Bob", "Alice")  // Size 2
uniqueNames.contains("Alice")  // true

// Set operations
val a = setOf(1, 2, 3)
val b = setOf(2, 3, 4)
a union b       // {1, 2, 3, 4}
a intersect b   // {2, 3}
a subtract b    // {1}
```

### Map

Key-value pairs with unique keys:

```kotlin
val capitals = mapOf(
    "UK" to "London",
    "France" to "Paris",
    "Germany" to "Berlin"
)

capitals["UK"]                   // "London"
capitals["Spain"]                // null
capitals.getOrDefault("Spain", "Unknown")  // "Unknown"

// Destructuring
for ((country, city) in capitals) {
    println("$country -> $city")
}
```

### Mutable Collection Patterns

```kotlin
val scores = mutableMapOf<String, Int>()
scores["Alice"] = 95
scores["Bob"] = 87
scores.getOrPut("Charlie") { 0 }  // Inserts 0 if absent, returns the value

val queue = ArrayDeque<String>()
queue.addLast("first")
queue.addLast("second")
queue.removeFirst()  // "first"
```

## Collection Operations

Kotlin provides a comprehensive set of functional operations on all collection types.

### Transformations

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

numbers.map { it * 2 }                // [2, 4, 6, 8, 10]
numbers.mapIndexed { i, n -> "$i:$n" } // ["0:1", "1:2", "2:3", ...]
numbers.mapNotNull { if (it > 3) it else null }  // [4, 5]

// flatMap — map + flatten
val nested = listOf(listOf(1, 2), listOf(3, 4))
nested.flatMap { it }                  // [1, 2, 3, 4]
nested.flatten()                       // Same result

// Associate — build maps from lists
val users = listOf("Alice", "Bob", "Charlie")
users.associateWith { it.length }      // {Alice=5, Bob=3, Charlie=7}
users.associateBy { it.first() }       // {A=Alice, B=Bob, C=Charlie}
```

### Filtering

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)

numbers.filter { it % 2 == 0 }           // [2, 4, 6]
numbers.filterNot { it % 2 == 0 }        // [1, 3, 5]
numbers.filterIsInstance<Int>()           // Type-based filter

// Partition — split into matching and non-matching
val (evens, odds) = numbers.partition { it % 2 == 0 }
// evens = [2, 4, 6], odds = [1, 3, 5]

// Take and drop
numbers.take(3)          // [1, 2, 3]
numbers.drop(3)          // [4, 5, 6]
numbers.takeWhile { it < 4 }  // [1, 2, 3]
numbers.dropWhile { it < 4 }  // [4, 5, 6]
```

### Grouping and Aggregation

```kotlin
data class Person(val name: String, val city: String, val age: Int)

val people = listOf(
    Person("Alice", "London", 30),
    Person("Bob", "London", 25),
    Person("Charlie", "Paris", 35),
    Person("Diana", "Paris", 28)
)

// groupBy — group elements by a key
people.groupBy { it.city }
// {London=[Alice, Bob], Paris=[Charlie, Diana]}

// Aggregation
val numbers = listOf(3, 1, 4, 1, 5, 9)
numbers.sum()           // 23
numbers.average()       // 3.83...
numbers.min()           // 1
numbers.max()           // 9
numbers.count { it > 3 }  // 3

// fold and reduce
numbers.fold(0) { acc, n -> acc + n }      // 23
numbers.reduce { acc, n -> acc + n }       // 23 (no initial value)
numbers.runningFold(0) { acc, n -> acc + n }  // [0, 3, 4, 8, 9, 14, 23]
```

### Sorting

```kotlin
val names = listOf("Charlie", "Alice", "Bob")

names.sorted()                         // [Alice, Bob, Charlie]
names.sortedDescending()               // [Charlie, Bob, Alice]
names.sortedBy { it.length }           // [Bob, Alice, Charlie]
names.sortedByDescending { it.length } // [Charlie, Alice, Bob]

// Custom comparator
names.sortedWith(compareBy<String> { it.length }.thenBy { it })
```

### Checking Conditions

```kotlin
val numbers = listOf(2, 4, 6, 8)

numbers.all { it % 2 == 0 }    // true — every element matches
numbers.any { it > 5 }         // true — at least one matches
numbers.none { it < 0 }        // true — no element matches
```

## Sequences

Sequences process elements lazily, one at a time through the entire chain, rather than creating intermediate collections at each step.

### Eager vs Lazy Evaluation

```kotlin
// Eager — creates an intermediate list at each step
val eagerResult = listOf(1, 2, 3, 4, 5)
    .map { println("map $it"); it * 2 }      // Processes ALL elements
    .filter { println("filter $it"); it > 4 } // Processes ALL elements
    .first()

// Lazy — processes elements one at a time
val lazyResult = listOf(1, 2, 3, 4, 5)
    .asSequence()
    .map { println("map $it"); it * 2 }
    .filter { println("filter $it"); it > 4 }
    .first()  // Stops after finding the first match
```

With sequences, the processing order is element-by-element (vertical), not operation-by-operation (horizontal):

| Order | Eager (List) | Lazy (Sequence) |
|-------|-------------|-----------------|
| Processing | map ALL, then filter ALL | map+filter item 1, then item 2, ... |
| Intermediate collections | One per operation | None |
| Short-circuit | Not possible | `first()`, `take()`, etc. stop early |

### Creating Sequences

```kotlin
// From collections
val fromList = listOf(1, 2, 3).asSequence()

// Generator function
val naturals = generateSequence(1) { it + 1 }  // Infinite sequence
val firstTen = naturals.take(10).toList()

// From a block (like a coroutine)
val fibonacci = sequence {
    var a = 0
    var b = 1
    while (true) {
        yield(a)
        val next = a + b
        a = b
        b = next
    }
}
fibonacci.take(8).toList()  // [0, 1, 1, 2, 3, 5, 8, 13]
```

### When to Use Sequences

| Scenario | Use |
|----------|-----|
| Small collections (< 100 elements) | `List` operations — overhead of sequence not worth it |
| Large collections with chained operations | `Sequence` — avoids intermediate allocations |
| Infinite or unbounded data | `Sequence` — only computes what is consumed |
| Single operation (just `map` or just `filter`) | `List` — no benefit from sequences |
| Multiple chained operations with early termination | `Sequence` — short-circuits |

### Terminal Operations

Sequences are lazy until a terminal operation forces evaluation:

```kotlin
val result = (1..1_000_000)
    .asSequence()
    .map { it * 2 }
    .filter { it % 3 == 0 }
    .take(10)      // Still lazy
    .toList()      // Terminal — triggers evaluation

// Common terminal operations: toList(), toSet(), first(), count(), sum(), forEach()
```

## Building Collections

Kotlin provides builder functions for constructing collections conditionally:

```kotlin
val list = buildList {
    add("always")
    if (condition) add("sometimes")
    addAll(otherList)
}

val map = buildMap {
    put("key1", "value1")
    if (includeExtra) put("key2", "value2")
}

val set = buildSet {
    add("a")
    add("b")
    addAll(otherSet)
}
```

## Key Takeaways

- Kotlin separates read-only (`List`, `Set`, `Map`) from mutable (`MutableList`, `MutableSet`, `MutableMap`) at the type level — prefer read-only interfaces
- Collection operations like `map`, `filter`, `flatMap`, `groupBy`, and `fold` are chainable and expressive — they cover most data transformation needs
- `partition` splits a collection into matching and non-matching elements in a single pass
- Sequences provide lazy evaluation, avoiding intermediate collection allocations — use them for large datasets or chained operations with early termination
- The `sequence { yield() }` builder creates infinite or custom sequences with coroutine-like syntax
- Builder functions (`buildList`, `buildMap`) provide clean conditional collection construction
