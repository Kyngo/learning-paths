---
title: "Functions and Lambdas"
weight: 2
---

# Functions and Lambdas

Kotlin's function system combines concise syntax with powerful features like default parameters, extension functions, and first-class lambdas. Functions are the primary building blocks for organising logic, and Kotlin makes them flexible enough to serve as both simple utilities and composable abstractions.

## Function Syntax

Functions are declared with the `fun` keyword. The return type follows the parameter list:

```kotlin
fun greet(name: String): String {
    return "Hello, $name!"
}

// Single-expression functions omit braces and return
fun greet(name: String): String = "Hello, $name!"

// Return type can be inferred for single-expression functions
fun greet(name: String) = "Hello, $name!"
```

Functions that return nothing meaningful use `Unit` (equivalent to `void`). The return type and `return` statement can be omitted:

```kotlin
fun log(message: String) {
    println("[LOG] $message")
}
// Equivalent to: fun log(message: String): Unit { ... }
```

## Default and Named Parameters

Default values eliminate the need for method overloads:

```kotlin
fun connect(
    host: String,
    port: Int = 5432,
    ssl: Boolean = true,
    timeout: Int = 30_000
): Connection {
    // ...
}

// Call with defaults
connect("db.example.com")

// Override specific parameters by name
connect("db.example.com", ssl = false)
connect("db.example.com", timeout = 60_000, port = 3306)
```

Named arguments make call sites self-documenting and allow skipping parameters with defaults in any order.

### Rules for Default Parameters

| Rule | Example |
|------|---------|
| Parameters with defaults can be omitted | `connect("host")` |
| Named arguments can be in any order | `connect(port = 80, host = "x")` |
| Positional arguments must precede named | `connect("host", port = 80)` ✓ |
| Mixing positional after named is not allowed | `connect(host = "x", 80)` ✗ |

## Extension Functions

Extension functions add behaviour to existing types without modifying their source code:

```kotlin
fun String.initials(): String =
    split(" ")
        .filter { it.isNotBlank() }
        .map { it.first().uppercase() }
        .joinToString("")

"John Doe".initials()  // "JD"
```

Extension functions are resolved statically (at compile time), not dynamically. They do not actually modify the class:

```kotlin
open class Shape
class Circle : Shape()

fun Shape.name() = "Shape"
fun Circle.name() = "Circle"

val shape: Shape = Circle()
println(shape.name())  // "Shape" — resolved by declared type, not runtime type
```

### Extension Properties

Extensions can also define properties (but they cannot store state — no backing fields):

```kotlin
val String.wordCount: Int
    get() = split(Regex("\\s+")).filter { it.isNotBlank() }.size

"Hello Kotlin world".wordCount  // 3
```

## Higher-Order Functions

A higher-order function takes a function as a parameter or returns one. This is the foundation of Kotlin's functional style:

```kotlin
fun <T> List<T>.customFilter(predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) result.add(item)
    }
    return result
}

val evens = listOf(1, 2, 3, 4, 5).customFilter { it % 2 == 0 }  // [2, 4]
```

The function type syntax is `(ParameterTypes) -> ReturnType`:

| Type Signature | Meaning |
|----------------|---------|
| `() -> Unit` | No parameters, no meaningful return |
| `(Int) -> Boolean` | Takes Int, returns Boolean |
| `(String, Int) -> String` | Takes String and Int, returns String |
| `Int.() -> String` | Extension function on Int returning String |

## Lambda Syntax

Lambdas are anonymous functions enclosed in braces:

```kotlin
val double = { x: Int -> x * 2 }
double(5)  // 10

// When the type is known, parameter types can be omitted
val numbers = listOf(1, 2, 3, 4)
val doubled = numbers.map { x -> x * 2 }

// Single parameter can be referenced as 'it'
val tripled = numbers.map { it * 3 }
```

### Trailing Lambda Convention

When the last parameter of a function is a lambda, it can be placed outside the parentheses:

```kotlin
// Standard call
numbers.filter({ it > 2 })

// Trailing lambda — preferred
numbers.filter { it > 2 }

// If the lambda is the only argument, parentheses can be omitted entirely
run { println("Hello") }
```

### Destructuring in Lambdas

```kotlin
val map = mapOf("a" to 1, "b" to 2)
map.forEach { (key, value) ->
    println("$key = $value")
}

val pairs = listOf(Pair("x", 1), Pair("y", 2))
pairs.map { (letter, number) -> "$letter:$number" }
```

## Inline Functions

The `inline` keyword copies the function body at each call site, eliminating the overhead of lambda object creation:

```kotlin
inline fun <T> measureTime(block: () -> T): T {
    val start = System.nanoTime()
    val result = block()
    val elapsed = System.nanoTime() - start
    println("Took ${elapsed / 1_000_000}ms")
    return result
}

// No lambda object created — code is inlined
val data = measureTime {
    loadFromNetwork()
}
```

### noinline and crossinline

When an inline function has multiple lambda parameters, you can opt specific ones out:

```kotlin
inline fun transaction(
    noinline onError: (Exception) -> Unit,   // Not inlined — can be stored
    crossinline action: () -> Unit           // Inlined but cannot use non-local return
) {
    try {
        action()
    } catch (e: Exception) {
        onError(e)
    }
}
```

| Modifier | Effect |
|----------|--------|
| `inline` | Function body + lambdas inlined at call site |
| `noinline` | Specific lambda parameter is not inlined |
| `crossinline` | Inlined but forbids non-local returns from the lambda |

## Scope Functions

Scope functions execute a block of code in the context of an object. They differ in how they reference the object and what they return:

```kotlin
// let — transform or null-check
val length = "Kotlin".let { it.length }  // 6

// run — execute a block with 'this' as receiver
val result = "Kotlin".run {
    println("Processing $this")
    length  // last expression is the return value
}

// with — same as run but called differently
val info = with("Kotlin") {
    "Language: $this, length: $length"
}

// apply — configure and return the same object
val request = Request().apply {
    url = "https://api.example.com"
    method = "GET"
    addHeader("Accept", "application/json")
}

// also — side effects, return the same object
val user = User("Alice").also {
    logger.info("Created user: ${it.name}")
}
```

### Scope Function Selection Guide

| Function | Object ref | Returns | Typical use |
|----------|-----------|---------|-------------|
| `let` | `it` | Lambda result | Null-safe transforms, mapping |
| `run` | `this` | Lambda result | Object computation |
| `with` | `this` | Lambda result | Grouping calls (non-null) |
| `apply` | `this` | Object itself | Object initialisation |
| `also` | `it` | Object itself | Side effects, logging |

## Function References

Functions can be referenced as values using `::`:

```kotlin
fun isPositive(n: Int) = n > 0

val positives = listOf(-1, 0, 2, -3, 4).filter(::isPositive)  // [2, 4]

// Method references
val lengths = listOf("a", "bb", "ccc").map(String::length)     // [1, 2, 3]

// Constructor references
data class User(val name: String)
val users = listOf("Alice", "Bob").map(::User)
```

## Local Functions

Functions can be declared inside other functions, giving them access to the outer function's parameters and variables:

```kotlin
fun validate(user: User) {
    fun requireNotBlank(value: String, fieldName: String) {
        require(value.isNotBlank()) { "$fieldName must not be blank" }
    }

    requireNotBlank(user.name, "name")
    requireNotBlank(user.email, "email")
}
```

## Key Takeaways

- Single-expression functions with `=` eliminate boilerplate for simple transformations
- Default and named parameters replace the need for method overloading
- Extension functions add behaviour to existing types without inheritance or decoration
- Higher-order functions and lambdas enable concise functional patterns — trailing lambda syntax keeps code readable
- Inline functions eliminate lambda allocation overhead for performance-critical code
- Scope functions (`let`, `run`, `apply`, `also`, `with`) provide contextual code blocks — choose based on whether you need `it` vs `this` and the desired return value
