---
title: "Modern Java"
weight: 13
---

# Modern Java (17–21+)

Java has evolved dramatically since version 8. Records, sealed classes, pattern matching, virtual threads, and text blocks make modern Java significantly more concise and expressive than its predecessor. This section covers the features from Java 17 through 21+ that every Java developer should use.

---

## Records (Java 16+)

Immutable data carriers — a one-line replacement for boilerplate POJOs:

```java
// Before: ~50 lines (fields, constructor, getters, equals, hashCode, toString)
// After:
public record User(long id, String name, String email) {}

var user = new User(1, "Alice", "alice@example.com");
user.name();    // "Alice" (accessor, not getName())
user.toString(); // "User[id=1, name=Alice, email=alice@example.com]"

// Compact constructor (validation)
public record User(long id, String name, String email) {
    public User {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("name required");
    }
}

// Custom methods
public record Point(double x, double y) {
    public double distanceTo(Point other) {
        return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
    }
}
```

### When to Use Records

| Use Records | Use Classes |
|-------------|------------|
| DTOs, value objects | Mutable state |
| API request/response types | Entities with identity |
| Configuration objects | Complex inheritance |
| Event payloads | Builder patterns |

---

## Sealed Classes (Java 17+)

Restrict which classes can extend a type — the algebraic data type equivalent:

```java
public sealed interface Shape
    permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}
```

The compiler now knows all possible subtypes. Combined with pattern matching, this enables exhaustive checks:

```java
public double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> 0.5 * t.base() * t.height();
        // No default needed — compiler knows this is exhaustive
    };
}
```

---

## Pattern Matching

### `instanceof` Pattern (Java 16+)

```java
// Before
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// After — casts and binds in one expression
if (obj instanceof String s) {
    System.out.println(s.length());
}

// With negation
if (!(obj instanceof String s)) {
    return;
}
// s is in scope here
```

### Switch Pattern Matching (Java 21+)

```java
static String format(Object obj) {
    return switch (obj) {
        case Integer i -> "int: %d".formatted(i);
        case String s  -> "string: %s".formatted(s);
        case null       -> "null";
        default        -> "unknown: %s".formatted(obj);
    };
}

// Guarded patterns
static String classify(Shape shape) {
    return switch (shape) {
        case Circle c when c.radius() > 100 -> "large circle";
        case Circle c -> "circle";
        case Rectangle r when r.width() == r.height() -> "square";
        case Rectangle r -> "rectangle";
        case Triangle t -> "triangle";
    };
}
```

---

## Text Blocks (Java 15+)

Multi-line string literals with automatic indentation:

```java
String json = """
        {
            "name": "Alice",
            "age": 30,
            "roles": ["admin", "user"]
        }
        """;

String sql = """
        SELECT u.name, u.email
        FROM users u
        JOIN orders o ON u.id = o.user_id
        WHERE o.status = 'active'
        ORDER BY u.name
        """;

String html = """
        <html>
            <body>
                <h1>%s</h1>
                <p>%s</p>
            </body>
        </html>
        """.formatted(title, body);
```

---

## Switch Expressions (Java 14+)

```java
// Before — verbose, error-prone (missing break = fallthrough)
String label;
switch (day) {
    case MONDAY: case FRIDAY: label = "work"; break;
    case SATURDAY: case SUNDAY: label = "rest"; break;
    default: label = "midweek";
}

// After — expression, no fallthrough, exhaustive
String label = switch (day) {
    case MONDAY, FRIDAY -> "work";
    case SATURDAY, SUNDAY -> "rest";
    default -> "midweek";
};
```

---

## Virtual Threads (Java 21+)

Lightweight threads managed by the JVM — millions of concurrent tasks without thread pool tuning:

```java
// Platform thread (OS thread, ~1 MB stack)
Thread.ofPlatform().start(() -> handleRequest());

// Virtual thread (~few KB, scheduled on a carrier thread pool)
Thread.ofVirtual().start(() -> handleRequest());

// Executor with virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            var result = httpClient.send(request);  // blocks virtual thread, not OS thread
            process(result);
        });
    }
}
```

### Virtual Threads vs Platform Threads

| | Platform Threads | Virtual Threads |
|-|-----------------|----------------|
| Stack size | ~1 MB | ~few KB |
| Count | Thousands | Millions |
| Scheduling | OS kernel | JVM (M:N) |
| Blocking I/O | Blocks OS thread | Parks virtual thread, frees carrier |
| Use case | CPU-bound | I/O-bound (HTTP, DB, file) |

**Virtual threads make blocking I/O efficient again.** No need for reactive/async frameworks for I/O-bound services — just write blocking code with virtual threads.

---

## Other Modern Features

### `var` — Local Variable Type Inference (Java 10+)

```java
var list = new ArrayList<String>();     // ArrayList<String>
var map = Map.of("key", "value");       // Map<String, String>
var user = userRepository.findById(id); // Optional<User>

// Rules: only for local variables, not fields or method parameters
// Use when the type is obvious from the right side
```

### Helpful NullPointerExceptions (Java 14+)

```
// Before
Exception in thread "main" java.lang.NullPointerException

// After
Exception in thread "main" java.lang.NullPointerException:
    Cannot invoke "String.length()" because "user.getName()" is null
```

### String Methods

```java
"  hello  ".strip()        // "hello" (Unicode-aware trim)
"hello".repeat(3)          // "hellohellohello"
"hello\nworld".lines()     // Stream<String>
"  ".isBlank()             // true
"Hello".formatted("World") // equivalent to String.format
```

### Collectors and Stream Improvements

```java
// Collectors.teeing (Java 12+) — two collectors in one pass
var result = stream.collect(Collectors.teeing(
    Collectors.counting(),
    Collectors.averagingDouble(Order::total),
    (count, avg) -> new Stats(count, avg)
));

// Stream.toList() (Java 16+) — unmodifiable list
var list = stream.filter(x -> x > 0).toList();
```

---

## Key Takeaways

- Records replace 50 lines of POJO boilerplate with one line. Use them for DTOs, value objects, and event payloads.
- Sealed classes + pattern matching give you exhaustive switch expressions — the compiler catches missed cases.
- Virtual threads (Java 21+) let you write simple blocking code that scales to millions of concurrent I/O operations.
- Text blocks eliminate string concatenation and escaping for multi-line SQL, JSON, and HTML.
- Switch expressions return values, require no `break`, and are exhaustive — always prefer them over statements.
- Modern Java (17+) is a different language from Java 8. Upgrade and use the new features.
