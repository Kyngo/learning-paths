---
title: "Object-Oriented Kotlin"
weight: 4
---

# Object-Oriented Kotlin

Kotlin builds on Java's object-oriented foundation but strips away ceremony and adds powerful features like data classes, sealed hierarchies, object declarations, and delegation. The result is a concise, expressive class system that favours composition over inheritance.

## Classes and Constructors

Kotlin classes have a primary constructor in the class header and can have secondary constructors in the body:

```kotlin
class User(val name: String, val email: String, var isActive: Boolean = true) {
    // Properties declared in the primary constructor
    // val = read-only, var = mutable

    init {
        require(name.isNotBlank()) { "Name must not be blank" }
        require(email.contains("@")) { "Invalid email" }
    }

    // Secondary constructor
    constructor(name: String) : this(name, "$name@example.com")
}

val user = User("Alice", "alice@example.com")
println(user.name)       // Alice
println(user.isActive)   // true
```

### Properties and Accessors

Properties can have custom getters and setters:

```kotlin
class Temperature(celsius: Double) {
    var celsius: Double = celsius
        set(value) {
            require(value >= -273.15) { "Below absolute zero" }
            field = value  // 'field' is the backing field
        }

    val fahrenheit: Double         // Computed property — no backing field
        get() = celsius * 9 / 5 + 32

    val description: String
        get() = when {
            celsius < 0 -> "Freezing"
            celsius < 20 -> "Cold"
            celsius < 30 -> "Warm"
            else -> "Hot"
        }
}
```

### Visibility Modifiers

| Modifier | Class member | Top-level |
|----------|-------------|-----------|
| `public` (default) | Visible everywhere | Visible everywhere |
| `private` | Visible inside the class | Visible inside the file |
| `protected` | Visible in class + subclasses | N/A |
| `internal` | Visible in the same module | Visible in the same module |

## Inheritance

Classes are `final` by default. Use `open` to allow inheritance:

```kotlin
open class Shape(val name: String) {
    open fun area(): Double = 0.0
    open fun describe(): String = "Shape: $name"
}

class Circle(val radius: Double) : Shape("Circle") {
    override fun area(): Double = Math.PI * radius * radius
    override fun describe(): String = "${super.describe()}, radius=$radius"
}

class Rectangle(val width: Double, val height: Double) : Shape("Rectangle") {
    override fun area(): Double = width * height
}
```

### Abstract Classes

```kotlin
abstract class Vehicle(val make: String) {
    abstract fun fuelType(): String          // Must be overridden
    open fun describe() = "$make (${fuelType()})"  // Can be overridden
    fun registration() = "REG-${make.take(3).uppercase()}"  // Cannot be overridden
}

class ElectricCar(make: String) : Vehicle(make) {
    override fun fuelType() = "Electric"
}
```

### Interfaces

Interfaces can declare abstract methods, default implementations, and abstract properties:

```kotlin
interface Printable {
    val label: String              // Abstract property
    fun print()                     // Abstract method
    fun preview() = "Preview: $label"  // Default implementation
}

interface Loggable {
    fun log() = println("Logged")
}

class Report(override val label: String) : Printable, Loggable {
    override fun print() = println("Printing: $label")
}
```

## Data Classes

Data classes automatically generate `equals()`, `hashCode()`, `toString()`, `copy()`, and `componentN()` functions from the properties declared in the primary constructor:

```kotlin
data class Point(val x: Double, val y: Double)

val p1 = Point(1.0, 2.0)
val p2 = Point(1.0, 2.0)

p1 == p2          // true (structural equality via generated equals)
p1.toString()     // "Point(x=1.0, y=2.0)"

// copy() with named parameters for selective modification
val p3 = p1.copy(y = 5.0)  // Point(x=1.0, y=5.0)

// Destructuring via componentN()
val (x, y) = p1
```

### Data Class Rules

| Rule | Detail |
|------|--------|
| Primary constructor must have at least one `val`/`var` parameter | `data class Empty` is not allowed |
| Only primary constructor properties participate in generated methods | Properties in the body are excluded |
| Cannot be `abstract`, `open`, `sealed`, or `inner` | Data classes are final |
| `copy()` is shallow | Nested mutable objects share references |

```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String
) {
    // This property is NOT included in equals/hashCode/toString
    var loginCount: Int = 0
}
```

## Sealed Classes and Interfaces

Sealed types restrict which classes can inherit from them. All direct subtypes must be defined in the same package (and ideally the same file). This enables exhaustive `when` expressions:

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

fun <T> handleResult(result: Result<T>) = when (result) {
    is Result.Success -> println("Data: ${result.data}")
    is Result.Error -> println("Error: ${result.message}")
    Result.Loading -> println("Loading...")
    // No else needed — compiler knows all subtypes
}
```

### Sealed Interfaces

Sealed interfaces allow subtypes to extend other classes:

```kotlin
sealed interface UiState {
    data object Loading : UiState
    data class Content(val items: List<String>) : UiState
    data class Error(val message: String) : UiState
}

// A class can implement a sealed interface and extend another class
class NetworkError(message: String, val code: Int) : Exception(message), UiState
```

### Sealed vs Enum

| Feature | Sealed class/interface | Enum |
|---------|----------------------|------|
| Instances | Multiple per subtype | One per constant |
| State | Each subtype can hold different data | Shared property set |
| Exhaustive when | Yes | Yes |
| Inheritance | Subtypes can have their own hierarchies | No further inheritance |
| Use case | Complex state modelling | Fixed set of simple constants |

## Object Declarations

Kotlin's `object` keyword declares a singleton — a class with exactly one instance:

```kotlin
object DatabaseConfig {
    val host = "localhost"
    val port = 5432

    fun connectionString() = "jdbc:postgresql://$host:$port/mydb"
}

// Usage — no instantiation needed
val url = DatabaseConfig.connectionString()
```

### Object Expressions (Anonymous Objects)

Object expressions create one-off instances, similar to Java's anonymous inner classes:

```kotlin
val comparator = object : Comparator<String> {
    override fun compare(a: String, b: String): Int = a.length - b.length
}

// Or implementing an interface inline
interface ClickListener {
    fun onClick(id: String)
}

val listener = object : ClickListener {
    override fun onClick(id: String) = println("Clicked: $id")
}
```

## Companion Objects

A companion object is tied to a class (not an instance). It is Kotlin's replacement for Java's static members:

```kotlin
class User private constructor(val id: Long, val name: String) {
    companion object {
        private var nextId = 0L

        fun create(name: String): User {
            return User(++nextId, name)
        }

        fun fromJson(json: String): User {
            // Parse JSON and create User
            return create(json)  // Simplified
        }
    }
}

val user = User.create("Alice")
// val user = User(1, "Alice")  // Error — constructor is private
```

Companion objects can implement interfaces:

```kotlin
interface Factory<T> {
    fun create(): T
}

class MyService private constructor() {
    companion object : Factory<MyService> {
        override fun create() = MyService()
    }
}

// Can be passed as a Factory<MyService>
fun <T> buildInstance(factory: Factory<T>): T = factory.create()
val service = buildInstance(MyService)
```

## Delegation (by Keyword)

Kotlin supports the delegation pattern at the language level using `by`. This lets you compose behaviour without inheritance:

### Interface Delegation

```kotlin
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println("[CONSOLE] $message")
}

class FileLogger(private val path: String) : Logger {
    override fun log(message: String) { /* write to file */ }
}

// Repository delegates Logger implementation to the injected instance
class Repository(logger: Logger) : Logger by logger {
    fun save(data: String) {
        log("Saving: $data")  // Calls the delegated logger
    }
}

val repo = Repository(ConsoleLogger())
repo.save("test")  // [CONSOLE] Saving: test
```

### Property Delegation

Kotlin allows delegating property access to another object using `by`:

```kotlin
import kotlin.properties.Delegates

// lazy — computed on first access, cached
val expensiveData: List<String> by lazy {
    println("Computing...")
    loadFromDatabase()
}

// observable — callback on every change
var status: String by Delegates.observable("idle") { _, old, new ->
    println("$old -> $new")
}

// vetoable — can reject changes
var age: Int by Delegates.vetoable(0) { _, _, new ->
    new >= 0  // Only accept non-negative values
}

// Map delegation — read properties from a map
class Config(map: Map<String, Any>) {
    val host: String by map
    val port: Int by map
}

val config = Config(mapOf("host" to "localhost", "port" to 5432))
config.host  // "localhost"
```

## Enum Classes

Enums define a fixed set of constants, optionally with properties and methods:

```kotlin
enum class Priority(val level: Int) {
    LOW(1),
    MEDIUM(2),
    HIGH(3),
    CRITICAL(4);

    fun isUrgent(): Boolean = level >= 3
}

Priority.HIGH.level      // 3
Priority.HIGH.isUrgent() // true
Priority.valueOf("LOW")  // Priority.LOW
Priority.entries          // [LOW, MEDIUM, HIGH, CRITICAL]
```

## Key Takeaways

- Classes are `final` by default — use `open` explicitly when designing for inheritance
- Data classes generate `equals`, `hashCode`, `toString`, `copy`, and destructuring — use them for value-carrying types
- Sealed classes/interfaces restrict inheritance to a known set of subtypes, enabling exhaustive `when` matching
- `object` declarations create singletons; companion objects replace Java's static members
- Interface delegation with `by` enables composition without boilerplate forwarding methods
- Property delegation (`lazy`, `observable`, `vetoable`, map delegation) reduces repetitive accessor patterns
