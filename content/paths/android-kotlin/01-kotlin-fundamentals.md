---
title: "Kotlin Fundamentals"
weight: 1
---

# Kotlin Fundamentals

Kotlin is a statically-typed language that runs on the JVM, designed to be concise, safe, and fully interoperable with Java. It eliminates entire categories of bugs through its type system, particularly its handling of null references through nullable types.

## Variables: val and var

Kotlin distinguishes between immutable and mutable bindings at the declaration site:

```kotlin
val maximumAttempts = 3      // Immutable — cannot be reassigned
var currentAttempt = 0       // Mutable — can be reassigned

currentAttempt += 1          // OK
// maximumAttempts = 5       // Compile error: Val cannot be reassigned
```

Prefer `val` over `var` whenever possible. The compiler enforces immutability, making code easier to reason about and safer in concurrent contexts.

### Type Inference and Annotations

Kotlin infers types from the right-hand side, but you can provide explicit annotations:

```kotlin
val name: String = "Kotlin"
val version: Double = 2.0
val isStable: Boolean = true
val year: Int = 2024
```

When the type cannot be inferred (e.g., a declaration without initialisation), annotations are required:

```kotlin
val score: Int       // Must declare type if not initialised here
score = 100          // Assigned later (only valid for local vals with guaranteed single assignment)
```

## Basic Types

Kotlin's type system has no primitive/wrapper distinction at the language level. The compiler optimises to JVM primitives where possible.

| Type | Size | Example |
|------|------|---------|
| `Byte` | 8-bit | `val b: Byte = 127` |
| `Short` | 16-bit | `val s: Short = 32767` |
| `Int` | 32-bit | `val i = 42` |
| `Long` | 64-bit | `val l = 100L` |
| `Float` | 32-bit | `val f = 3.14f` |
| `Double` | 64-bit | `val d = 3.14` |
| `Boolean` | — | `val flag = true` |
| `Char` | 16-bit | `val c = 'A'` |

Kotlin does not perform implicit numeric conversions:

```kotlin
val intValue: Int = 42
val longValue: Long = intValue.toLong()  // Explicit conversion required

// val wrong: Long = intValue  // Compile error!
```

## String Templates

String templates embed expressions directly inside strings using `$` or `${}`:

```kotlin
val language = "Kotlin"
val version = 2.0

println("Learning $language")                     // Simple reference
println("Version ${version + 0.1}")               // Expression
println("Name length: ${language.length}")         // Property access
println("Upper: ${language.uppercase()}")          // Method call
```

Multi-line strings use triple quotes and preserve indentation:

```kotlin
val json = """
    {
        "name": "$language",
        "version": $version
    }
""".trimIndent()
```

## Null Safety

Kotlin's type system distinguishes between nullable and non-nullable references. A regular type like `String` can never hold `null` — you must explicitly opt in with `String?`.

```kotlin
var name: String = "Kotlin"
// name = null               // Compile error: Null cannot be a value of a non-null type

var nickname: String? = "K"
nickname = null              // OK — type is nullable
```

### Safe Call Operator (?.)

The safe call returns `null` if the receiver is null, instead of throwing a NullPointerException:

```kotlin
val length: Int? = nickname?.length    // null if nickname is null
val upper: String? = nickname?.uppercase()

// Chaining safe calls
val firstChar: Char? = nickname?.uppercase()?.first()
```

### Elvis Operator (?:)

Provides a default value when the left side is null:

```kotlin
val displayName: String = nickname ?: "Anonymous"
val length: Int = nickname?.length ?: 0

// Can also throw or return on the right side
val validName: String = nickname ?: throw IllegalArgumentException("Name required")
```

### Not-Null Assertion (!!)

Forces a nullable type to non-null. Throws `NullPointerException` if the value is null:

```kotlin
val definitelyNotNull: String = nickname!!  // Throws NPE if null
```

Avoid `!!` in production code. It defeats the purpose of null safety. Use it only in tests or when you have a guarantee the compiler cannot verify.

### Scope Functions for Null Handling

Kotlin's scope functions are particularly useful with nullable types:

```kotlin
// let — execute block only if non-null
nickname?.let { name ->
    println("Nickname is $name")
}

// also — perform side effects, return the original value
val validated = nickname?.also { println("Validating: $it") }

// apply — configure an object, return the object itself
val config = Config().apply {
    host = "localhost"
    port = 8080
}
```

| Function | Receiver | Returns | Use case |
|----------|----------|---------|----------|
| `let` | `it` | Lambda result | Transform or null-check |
| `also` | `it` | Original object | Side effects (logging, validation) |
| `apply` | `this` | Original object | Object configuration |
| `run` | `this` | Lambda result | Object transformation |
| `with` | `this` | Lambda result | Grouping calls on an object |

## When Expressions

`when` is Kotlin's replacement for `switch`, and it is far more powerful. It can match values, ranges, types, and arbitrary conditions:

```kotlin
val statusCode = 404

val message = when (statusCode) {
    200 -> "OK"
    301, 302 -> "Redirect"
    in 400..499 -> "Client error"
    in 500..599 -> "Server error"
    else -> "Unknown"
}
```

### Type Checking with when

```kotlin
fun describe(obj: Any): String = when (obj) {
    is String -> "String of length ${obj.length}"   // Smart cast — obj is String here
    is Int -> "Integer: $obj"
    is List<*> -> "List of size ${obj.size}"
    else -> "Unknown type"
}
```

### when Without an Argument

Used as a cleaner alternative to `if-else` chains:

```kotlin
fun categorise(temperature: Int): String = when {
    temperature < 0 -> "Freezing"
    temperature < 15 -> "Cold"
    temperature < 25 -> "Comfortable"
    temperature < 35 -> "Hot"
    else -> "Extreme"
}
```

### Exhaustive when

When used as an expression (assigned to a variable or returned), `when` must be exhaustive. With sealed classes or enums, the compiler verifies all cases are handled:

```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST }

val label = when (direction) {
    Direction.NORTH -> "Up"
    Direction.SOUTH -> "Down"
    Direction.EAST -> "Right"
    Direction.WEST -> "Left"
    // No else needed — all cases covered
}
```

## Ranges and Progressions

Ranges define intervals and are commonly used with `in`, loops, and `when`:

```kotlin
val singleDigits = 1..9           // Closed range: 1 to 9 inclusive
val indices = 0 until 10          // Half-open: 0 to 9
val countdown = 10 downTo 1       // Descending: 10 to 1
val evens = 0..20 step 2          // With step: 0, 2, 4, ..., 20

// Membership check
val inRange = 5 in 1..10          // true
val outOfRange = 15 !in 1..10     // true

// Iteration
for (i in 1..5) {
    print("$i ")                  // 1 2 3 4 5
}

for (i in 5 downTo 1 step 2) {
    print("$i ")                  // 5 3 1
}
```

Ranges also work with `Char` and `Comparable` types:

```kotlin
val letters = 'a'..'z'
val isLower = 'g' in letters      // true

val dateRange = LocalDate.of(2024, 1, 1)..LocalDate.of(2024, 12, 31)
```

## Control Flow

### If as an Expression

In Kotlin, `if` is an expression that returns a value:

```kotlin
val max = if (a > b) a else b

val description = if (temperature > 30) {
    "Hot day"
} else if (temperature > 15) {
    "Pleasant"
} else {
    "Cold"
}
```

### For Loops

```kotlin
for (i in 1..5) {
    println(i)
}

for (item in collection) {
    println(item)
}

// With index
for ((index, value) in collection.withIndex()) {
    println("$index: $value")
}
```

### While and Do-While

```kotlin
var count = 5
while (count > 0) {
    println(count)
    count--
}

do {
    val input = readLine()
} while (input != "quit")
```

## Type Checks and Smart Casts

The `is` operator checks types, and Kotlin automatically casts after a successful check:

```kotlin
fun processInput(input: Any) {
    if (input is String) {
        // Smart cast — input is automatically treated as String here
        println(input.uppercase())
        println("Length: ${input.length}")
    }
    if (input is Int && input > 0) {
        println("Positive number: $input")
    }
}
```

Explicit casting uses `as` (unsafe) or `as?` (safe):

```kotlin
val str: String = obj as String        // Throws ClassCastException if wrong type
val strOrNull: String? = obj as? String // Returns null if wrong type
```

## Key Takeaways

- Use `val` by default; only use `var` when mutation is genuinely needed
- Kotlin's null safety system (`?`, `?.`, `?:`, `!!`) eliminates NullPointerException at compile time — embrace it rather than using `!!`
- String templates with `$` and `${}` are preferred over concatenation
- `when` expressions are exhaustive when used as expressions and support ranges, types, and arbitrary conditions
- Ranges (`..`, `until`, `downTo`, `step`) provide expressive iteration and membership checks
- Smart casts automatically narrow types after `is` checks, reducing boilerplate
