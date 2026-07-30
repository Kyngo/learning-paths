---
title: "Java: Syntax, Types, and Operators"
weight: 2
---

## Primitive Types

Java has 8 primitive types — they live on the stack, not the heap.

| Type | Size | Range | Default | Wrapper |
|------|------|-------|---------|---------|
| `byte` | 8 bits | -128 to 127 | 0 | `Byte` |
| `short` | 16 bits | -32,768 to 32,767 | 0 | `Short` |
| `int` | 32 bits | -2³¹ to 2³¹-1 | 0 | `Integer` |
| `long` | 64 bits | -2⁶³ to 2⁶³-1 | 0L | `Long` |
| `float` | 32 bits | ±3.4×10³⁸ | 0.0f | `Float` |
| `double` | 64 bits | ±1.7×10³⁰⁸ | 0.0d | `Double` |
| `char` | 16 bits | 0 to 65,535 (Unicode) | '\u0000' | `Character` |
| `boolean` | JVM-specific | true/false | false | `Boolean` |

```java
// Literals
int decimal = 42;
int hex = 0x2A;           // 42 in hex
int binary = 0b101010;    // 42 in binary
long big = 10_000_000L;   // Underscores for readability
double pi = 3.14159;
float f = 3.14f;          // Must suffix with f
char c = 'A';
char unicode = '\u0041';  // Also 'A'

// Overflow wraps silently!
int max = Integer.MAX_VALUE;  // 2,147,483,647
int overflow = max + 1;       // -2,147,483,648 (no exception!)
```

### Autoboxing and Unboxing

```java
// Autoboxing: primitive → wrapper (automatic)
Integer boxed = 42;           // int → Integer
List<Integer> list = new ArrayList<>();
list.add(5);                  // int → Integer (autoboxed)

// Unboxing: wrapper → primitive (automatic)
int unboxed = boxed;          // Integer → int
int sum = boxed + 10;         // Unbox, add, result is int

// DANGER: NullPointerException on unboxing null
Integer nullable = null;
int crash = nullable;         // NullPointerException!

// Integer cache: -128 to 127 are cached
Integer a = 127;
Integer b = 127;
System.out.println(a == b);   // true (same cached object)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);   // false (different objects!)
System.out.println(c.equals(d)); // true (value comparison)
```

---

## Strings

```java
// Strings are immutable objects
String s = "Hello";
s.toUpperCase();        // Returns NEW string; s is unchanged
String upper = s.toUpperCase();  // "HELLO"

// String pool (interning)
String a = "hello";     // Goes to string pool
String b = "hello";     // Same reference from pool
String c = new String("hello");  // New object on heap (avoid this)
a == b;     // true (same reference)
a == c;     // false (different objects)
a.equals(c); // true (same content) — ALWAYS use .equals() for strings

// StringBuilder for concatenation in loops
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append("item ").append(i).append("\n");
}
String result = sb.toString();

// Text blocks (Java 13+)
String json = """
        {
            "name": "Alice",
            "age": 30
        }
        """;

// Useful methods
"Hello World".substring(0, 5);     // "Hello"
"Hello World".split(" ");          // ["Hello", "World"]
"  spaces  ".strip();              // "spaces" (Java 11+)
"Hello".contains("ell");           // true
"Hello".startsWith("He");          // true
String.format("Name: %s, Age: %d", name, age);
"Hello %s".formatted("World");     // Java 15+
```

---

## Type Inference (var)

```java
// Local variable type inference (Java 10+)
var list = new ArrayList<String>();  // Inferred as ArrayList<String>
var map = Map.of("key", "value");   // Inferred as Map<String, String>
var stream = list.stream();         // Inferred as Stream<String>

// Where var CANNOT be used:
// - Method parameters
// - Return types
// - Fields (instance/class variables)
// - When initializer is null

// Good uses
var connection = DriverManager.getConnection(url);  // Type is obvious
var entries = map.entrySet();  // Avoids verbose Set<Map.Entry<K,V>>

// Bad uses (reduces readability)
var result = process(data);  // What type is result?
var x = 42;                  // Just use int
```

---

## Arrays

```java
// Declaration and initialization
int[] numbers = new int[5];           // [0, 0, 0, 0, 0]
int[] primes = {2, 3, 5, 7, 11};     // Literal initialization
String[] names = new String[3];       // [null, null, null]

// Multi-dimensional
int[][] matrix = new int[3][4];       // 3 rows, 4 columns
int[][] jagged = {                    // Rows can have different lengths
    {1, 2, 3},
    {4, 5},
    {6, 7, 8, 9}
};

// Arrays are objects with fixed size
numbers.length;  // 5 (field, not method)
// numbers[5];   // ArrayIndexOutOfBoundsException

// Utility methods
import java.util.Arrays;
Arrays.sort(numbers);
Arrays.fill(numbers, 0);
int[] copy = Arrays.copyOf(numbers, 10);  // Extends with zeros
Arrays.equals(a, b);                       // Content comparison
Arrays.toString(numbers);                  // "[0, 0, 0, 0, 0]"
```

---

## Control Flow

```java
// Enhanced switch (Java 14+) — expression form
String dayType = switch (day) {
    case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
    case SATURDAY, SUNDAY -> "Weekend";
};

// Switch with blocks
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY -> 7;
    case THURSDAY, SATURDAY -> 8;
    case WEDNESDAY -> 9;
    default -> throw new IllegalArgumentException("Invalid day: " + day);
};

// Pattern matching for instanceof (Java 16+)
if (obj instanceof String s) {
    // s is already cast to String — no explicit cast needed
    System.out.println(s.length());
}

// Pattern matching in switch (Java 21+)
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "Positive integer: " + i;
        case Integer i -> "Non-positive integer: " + i;
        case String s -> "String of length " + s.length();
        case null -> "null";
        default -> "Unknown: " + obj.getClass().getName();
    };
}

// For-each
for (String name : names) {
    System.out.println(name);
}

// Labeled break (for nested loops)
outer:
for (int i = 0; i < rows; i++) {
    for (int j = 0; j < cols; j++) {
        if (matrix[i][j] == target) {
            found = true;
            break outer;  // Breaks out of BOTH loops
        }
    }
}
```

---

## Records (Java 16+)

```java
// Records: immutable data carriers with auto-generated methods
public record Point(double x, double y) {
    // Auto-generated: constructor, getters (x(), y()), equals(), hashCode(), toString()
    
    // Compact constructor for validation
    public Point {
        if (Double.isNaN(x) || Double.isNaN(y)) {
            throw new IllegalArgumentException("Coordinates cannot be NaN");
        }
    }
    
    // Additional methods
    public double distanceTo(Point other) {
        return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
    }
}

// Usage
var p1 = new Point(3, 4);
var p2 = new Point(0, 0);
p1.x();              // 3.0 (accessor, not getX())
p1.distanceTo(p2);   // 5.0
p1.toString();       // "Point[x=3.0, y=4.0]"

// Records in pattern matching
record Range(int min, int max) {}

static String classify(Range r) {
    return switch (r) {
        case Range(int min, int max) when min == max -> "Single value: " + min;
        case Range(int min, int max) when max - min < 10 -> "Narrow range";
        case Range(int min, int max) -> "Wide range: " + min + " to " + max;
    };
}
```

---

## Sealed Classes (Java 17+)

```java
// Sealed: restricts which classes can extend/implement
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}

public record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}

public record Triangle(double base, double height) implements Shape {
    public double area() { return 0.5 * base * height; }
}

// Exhaustive pattern matching — compiler knows all subtypes
String describe(Shape shape) {
    return switch (shape) {
        case Circle c -> "Circle with radius " + c.radius();
        case Rectangle r -> "Rectangle " + r.width() + "x" + r.height();
        case Triangle t -> "Triangle with base " + t.base();
        // No default needed — sealed type is exhaustive
    };
}
```

---

## Hypothetical Use Case: Type-Safe Configuration

```java
public sealed interface ConfigValue permits StringValue, IntValue, BoolValue, ListValue {
    String key();
}

public record StringValue(String key, String value) implements ConfigValue {}
public record IntValue(String key, int value) implements ConfigValue {}
public record BoolValue(String key, boolean value) implements ConfigValue {}
public record ListValue(String key, List<String> values) implements ConfigValue {}

public class Config {
    private final Map<String, ConfigValue> entries = new HashMap<>();
    
    public void set(ConfigValue value) {
        entries.put(value.key(), value);
    }
    
    public String getString(String key) {
        return switch (entries.get(key)) {
            case StringValue sv -> sv.value();
            case IntValue iv -> String.valueOf(iv.value());
            case BoolValue bv -> String.valueOf(bv.value());
            case ListValue lv -> String.join(",", lv.values());
            case null -> throw new NoSuchElementException("Key not found: " + key);
        };
    }
}
```

---

## Key Takeaways

1. **Primitives live on the stack** — wrappers are objects on the heap (autoboxing has cost)
2. **Always use `.equals()` for object comparison** — `==` compares references
3. **Strings are immutable** — use `StringBuilder` for repeated concatenation
4. **`var` is for local variables only** — use when the type is obvious from context
5. **Records** replace boilerplate data classes — immutable, with auto-generated methods
6. **Sealed types + pattern matching** enable exhaustive, type-safe branching
7. **Integer cache** (-128 to 127) means `==` works for small boxed integers but fails for larger ones
