---
title: "Kotlin Patterns and Ecosystem"
weight: 6
---

# Kotlin Patterns and Ecosystem

Beyond the core language, Kotlin offers powerful patterns for building domain-specific languages, handling errors gracefully, and sharing code across platforms. This section covers DSLs, type-safe builders, the Result type, value classes, Kotlin Multiplatform, and kotlinx.serialization.

## Domain-Specific Languages (DSLs)

Kotlin's combination of extension functions, lambdas with receivers, and operator overloading makes it ideal for building DSLs — small, specialised languages embedded in Kotlin:

```kotlin
// A simple HTML DSL
fun html(block: HtmlBuilder.() -> Unit): String {
    val builder = HtmlBuilder()
    builder.block()
    return builder.build()
}

class HtmlBuilder {
    private val elements = mutableListOf<String>()

    fun head(block: HeadBuilder.() -> Unit) {
        val head = HeadBuilder()
        head.block()
        elements.add("<head>${head.build()}</head>")
    }

    fun body(block: BodyBuilder.() -> Unit) {
        val body = BodyBuilder()
        body.block()
        elements.add("<body>${body.build()}</body>")
    }

    fun build() = "<html>${elements.joinToString("")}</html>"
}
```

Usage reads like a declarative specification:

```kotlin
val page = html {
    head {
        title("My Page")
    }
    body {
        h1("Hello, Kotlin DSL!")
        p("This is a paragraph.")
    }
}
```

### Lambda with Receiver

The key mechanism behind DSLs is the lambda with receiver. The receiver becomes `this` inside the lambda:

```kotlin
// Regular lambda: (StringBuilder) -> Unit
// Lambda with receiver: StringBuilder.() -> Unit

fun buildString(block: StringBuilder.() -> Unit): String {
    val sb = StringBuilder()
    sb.block()       // 'sb' becomes 'this' inside block
    return sb.toString()
}

val result = buildString {
    append("Hello")   // 'this' is the StringBuilder
    append(", ")
    append("World")
}
```

### @DslMarker

To prevent accidental access to outer receivers in nested DSLs, use `@DslMarker`:

```kotlin
@DslMarker
annotation class HtmlDsl

@HtmlDsl
class TableBuilder { /* ... */ }

@HtmlDsl
class RowBuilder { /* ... */ }

// Inside RowBuilder, you cannot accidentally call TableBuilder methods
```

## Type-Safe Builders

Type-safe builders are DSLs applied to configuration and construction patterns. Kotlin's standard library uses this pattern extensively:

```kotlin
// buildList — from the standard library
val items = buildList {
    add("first")
    addAll(listOf("second", "third"))
    if (includeExtra) add("extra")
}

// buildMap
val config = buildMap<String, Any> {
    put("host", "localhost")
    put("port", 5432)
    putAll(defaults)
}

// buildString
val csv = buildString {
    appendLine("name,age,city")
    for (user in users) {
        appendLine("${user.name},${user.age},${user.city}")
    }
}
```

### Custom Builder Example

```kotlin
class Route(val method: String, val path: String, val handler: () -> String)

class Router {
    private val routes = mutableListOf<Route>()

    fun get(path: String, handler: () -> String) {
        routes.add(Route("GET", path, handler))
    }

    fun post(path: String, handler: () -> String) {
        routes.add(Route("POST", path, handler))
    }

    fun build(): List<Route> = routes.toList()
}

fun router(block: Router.() -> Unit): List<Route> {
    val router = Router()
    router.block()
    return router.build()
}

// Usage
val routes = router {
    get("/users") { "user list" }
    get("/users/{id}") { "user detail" }
    post("/users") { "create user" }
}
```

## Result Type

`Result<T>` wraps a successful value or a failure, providing a functional alternative to try-catch:

```kotlin
fun parseJson(json: String): Result<Config> = runCatching {
    parser.parse(json)
}

// Handle the result
val config = parseJson(input)
    .map { it.copy(validated = true) }
    .getOrElse { default ->
        println("Parse failed: ${default.message}")
        Config.default()
    }

// Chain operations
fun loadConfig(path: String): Result<Config> = runCatching { readFile(path) }
    .mapCatching { parseJson(it) }
    .mapCatching { validate(it) }
    .onSuccess { println("Config loaded: ${it.name}") }
    .onFailure { println("Failed: ${it.message}") }
```

| Method | Purpose |
|--------|---------|
| `runCatching { }` | Execute block, wrap in Result |
| `getOrNull()` | Returns value or null |
| `getOrDefault(value)` | Returns value or the default |
| `getOrElse { }` | Returns value or computes fallback |
| `getOrThrow()` | Returns value or throws the exception |
| `map { }` | Transform success value |
| `mapCatching { }` | Transform, catching exceptions |
| `recover { }` | Transform failure into success |
| `onSuccess { }` | Side effect on success |
| `onFailure { }` | Side effect on failure |
| `fold(onSuccess, onFailure)` | Handle both cases |

## Inline Classes (Value Classes)

Value classes wrap a single value with zero runtime overhead. They provide type safety without the cost of object allocation:

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email: $value" }
    }

    fun domain(): String = value.substringAfter("@")
}
```

At compile time, the wrapper is erased — `UserId` becomes a plain `Long` in the bytecode. But at the source level, the type system prevents mixing up IDs:

```kotlin
fun findUser(id: UserId): User { /* ... */ }
fun findOrder(id: OrderId): Order { /* ... */ }

val userId = UserId(42)
val orderId = OrderId(42)

findUser(userId)    // OK
// findUser(orderId) // Compile error — type mismatch
```

### When to Use Value Classes

| Scenario | Value class? |
|----------|-------------|
| Wrapping an ID, email, URL, or other typed string/number | Yes |
| Adding validation to a primitive-like value | Yes |
| Replacing type aliases when you need enforcement | Yes |
| Wrapping multiple values | No — use a data class |
| Needing identity (reference equality) | No — value classes have no identity |

## Kotlin Multiplatform (KMP)

Kotlin Multiplatform lets you share Kotlin code across Android, iOS, desktop, and web while keeping platform-specific code separate:

```text
shared/
├── commonMain/          # Shared Kotlin code
│   └── kotlin/
│       ├── data/
│       │   └── UserRepository.kt
│       └── domain/
│           └── User.kt
├── androidMain/         # Android-specific implementations
│   └── kotlin/
│       └── Platform.kt
└── iosMain/             # iOS-specific implementations
    └── kotlin/
        └── Platform.kt
```

### expect/actual Pattern

Declare an expected interface in common code, and provide actual implementations per platform:

```kotlin
// commonMain — declaration
expect class Platform() {
    val name: String
    fun currentTimeMillis(): Long
}

// androidMain — implementation
actual class Platform actual constructor() {
    actual val name: String = "Android ${android.os.Build.VERSION.SDK_INT}"
    actual fun currentTimeMillis(): Long = System.currentTimeMillis()
}

// iosMain — implementation
actual class Platform actual constructor() {
    actual val name: String = UIDevice.currentDevice.systemName()
    actual fun currentTimeMillis(): Long = NSDate().timeIntervalSince1970.toLong() * 1000
}
```

### What to Share

| Layer | Share? | Notes |
|-------|--------|-------|
| Domain models | Yes | Data classes, enums, sealed classes |
| Business logic | Yes | Validation, computation, state machines |
| Networking | Yes | Using Ktor (multiplatform HTTP client) |
| Serialization | Yes | kotlinx.serialization works on all platforms |
| UI | Partially | Compose Multiplatform for Android/desktop/web; SwiftUI for iOS |
| Platform APIs | No | Use expect/actual for camera, sensors, etc. |

## Kotlinx Serialization

`kotlinx.serialization` is Kotlin's official serialization library. It uses compile-time code generation (no reflection) and supports JSON, Protobuf, CBOR, and other formats:

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class User(
    val id: Long,
    val name: String,
    val email: String,
    @SerialName("is_active")         // Map JSON key to property name
    val isActive: Boolean = true,
    val roles: List<String> = emptyList()
)

// Encode
val json = Json.encodeToString(User(1, "Alice", "alice@example.com"))
// {"id":1,"name":"Alice","email":"alice@example.com","is_active":true,"roles":[]}

// Decode
val user = Json.decodeFromString<User>(jsonString)
```

### JSON Configuration

```kotlin
val json = Json {
    ignoreUnknownKeys = true        // Don't fail on extra JSON fields
    isLenient = true                // Allow unquoted keys, trailing commas
    prettyPrint = true              // Formatted output
    encodeDefaults = false          // Omit properties with default values
    coerceInputValues = true        // Use default for null in non-null fields
}
```

### Polymorphic Serialization

Sealed class hierarchies are serialized with a type discriminator:

```kotlin
@Serializable
sealed class ApiResponse {
    @Serializable
    @SerialName("success")
    data class Success(val data: String) : ApiResponse()

    @Serializable
    @SerialName("error")
    data class Error(val message: String, val code: Int) : ApiResponse()
}

val response: ApiResponse = ApiResponse.Success("hello")
val jsonStr = Json.encodeToString(response)
// {"type":"success","data":"hello"}
```

### Comparison with Other Serializers

| Feature | kotlinx.serialization | Gson | Moshi |
|---------|----------------------|------|-------|
| Code generation | Compile-time (KSP) | Reflection | Reflection or codegen |
| Kotlin-first | Yes | No (Java) | Partial |
| Null safety | Respects Kotlin types | Ignores | Partial |
| Multiplatform | Yes | JVM only | JVM only |
| Sealed classes | Built-in support | Manual | Manual |
| Default values | Respected | Ignored | Respected |

## Key Takeaways

- Kotlin DSLs leverage lambdas with receivers to create readable, type-safe configuration code — the pattern is used extensively in Compose, Ktor, and Gradle
- The `Result<T>` type provides a functional approach to error handling with chainable operations like `map`, `recover`, and `fold`
- Value classes (`@JvmInline value class`) add type safety around primitives with zero runtime overhead — use them for IDs, emails, and domain-specific wrappers
- Kotlin Multiplatform (KMP) shares business logic across Android, iOS, desktop, and web using `expect`/`actual` for platform-specific code
- `kotlinx.serialization` is the preferred serialization library for Kotlin — it respects Kotlin's type system, supports sealed hierarchies, and works across all KMP targets
- Type-safe builders (`buildList`, `buildMap`, custom builders) are idiomatic Kotlin for constructing complex objects declaratively
