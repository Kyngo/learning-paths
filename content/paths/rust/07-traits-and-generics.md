---
title: "Traits & Generics"
weight: 7
---

# Traits & Generics

Traits define shared behaviour. Generics enable type-safe code that works across multiple types. Together they form Rust's mechanism for polymorphism — replacing inheritance and interfaces from object-oriented languages.

---

## Traits

A trait defines a set of methods a type can implement:

```rust
trait Summary {
    fn summarize(&self) -> String;

    // Default implementation (can be overridden)
    fn preview(&self) -> String {
        format!("{}...", &self.summarize()[..20])
    }
}
```

### Implementing a Trait

```rust
struct Article {
    title: String,
    author: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}, by {}", self.title, self.author)
    }
    // preview() uses the default implementation
}

let article = Article { /* ... */ };
println!("{}", article.summarize());
```

### Orphan Rule

You can implement a trait for a type only if either the trait or the type is defined in your crate. You cannot implement a foreign trait for a foreign type.

```rust
// OK — your trait, foreign type
impl Summary for Vec<String> { ... }

// OK — foreign trait, your type
impl Display for Article { ... }

// ERROR — foreign trait, foreign type
// impl Display for Vec<String> { ... }
```

Workaround: the **newtype pattern** — wrap the foreign type in your own struct.

---

## Common Standard Traits

| Trait | Purpose | Method |
|-------|---------|--------|
| `Display` | User-facing string (`{}`) | `fn fmt(&self, f: &mut Formatter) -> fmt::Result` |
| `Debug` | Developer string (`{:?}`) | Typically `#[derive(Debug)]` |
| `Clone` | Explicit deep copy | `fn clone(&self) -> Self` |
| `Copy` | Implicit bitwise copy | Marker trait (derive) |
| `PartialEq` / `Eq` | Equality (`==`) | `fn eq(&self, other: &Self) -> bool` |
| `PartialOrd` / `Ord` | Ordering (`<`, `>`) | `fn cmp(&self, other: &Self) -> Ordering` |
| `Hash` | Hashable | `fn hash<H: Hasher>(&self, state: &mut H)` |
| `Default` | Default value | `fn default() -> Self` |
| `From` / `Into` | Type conversion | `fn from(val: T) -> Self` |
| `TryFrom` / `TryInto` | Fallible conversion | `fn try_from(val: T) -> Result<Self, Error>` |
| `Iterator` | Iteration | `fn next(&mut self) -> Option<Item>` |
| `Drop` | Destructor | `fn drop(&mut self)` |
| `Deref` / `DerefMut` | Smart pointer dereferencing | `fn deref(&self) -> &Target` |
| `AsRef` / `AsMut` | Cheap reference conversion | `fn as_ref(&self) -> &T` |
| `Send` | Safe to transfer across threads | Marker trait (auto-implemented) |
| `Sync` | Safe to share across threads via &T | Marker trait (auto-implemented) |

---

## Generics

### Generic Functions

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut max = &list[0];
    for item in &list[1..] {
        if item > max {
            max = item;
        }
    }
    max
}

largest(&[1, 5, 3]);            // &5
largest(&["alpha", "beta"]);    // &"beta"
```

### Generic Structs

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point { x, y }
    }
}

// Implement methods only for specific types
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

### Generic Enums

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}

enum Option<T> {
    Some(T),
    None,
}
```

---

## Trait Bounds

Restrict generic types to those implementing certain traits:

```rust
// Bound syntax
fn print_summary<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}

// where clause (cleaner for multiple bounds)
fn process<T>(item: &T) -> String
where
    T: Summary + Display + Clone,
{
    format!("{}: {}", item, item.summarize())
}

// impl Trait syntax (simpler)
fn print_summary(item: &impl Summary) {
    println!("{}", item.summarize());
}
```

### Multiple Bounds

```rust
fn compare_and_display<T: PartialOrd + Display>(a: &T, b: &T) {
    if a > b {
        println!("{a} is greater");
    } else {
        println!("{b} is greater");
    }
}
```

---

## Trait Objects (Dynamic Dispatch)

When you need heterogeneous collections or runtime polymorphism:

```rust
// Static dispatch (monomorphisation) — each type gets its own compiled function
fn notify(item: &impl Summary) { ... }

// Dynamic dispatch (vtable) — single function, resolved at runtime
fn notify(item: &dyn Summary) { ... }

// Heterogeneous collection
let items: Vec<Box<dyn Summary>> = vec![
    Box::new(article),
    Box::new(tweet),
    Box::new(podcast),
];

for item in &items {
    println!("{}", item.summarize());
}
```

### Static vs Dynamic Dispatch

| | Static (`impl Trait`) | Dynamic (`dyn Trait`) |
|-|----------------------|----------------------|
| Resolution | Compile time | Runtime (vtable) |
| Performance | Zero cost (monomorphised) | Small overhead (indirection) |
| Binary size | Larger (duplicated code) | Smaller |
| Heterogeneous collections | ✗ | ✓ |
| Use when | Performance matters, types known | Need type erasure or plugins |

### Object Safety

A trait can be used as a trait object (`dyn Trait`) only if it is "object safe":
- No methods that return `Self`
- No generic methods
- All methods have a `self` receiver

---

## Associated Types

Alternative to generic parameters on traits — when there is exactly one implementation per type:

```rust
trait Iterator {
    type Item;  // associated type
    fn next(&mut self) -> Option<Self::Item>;
}

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> { ... }
}
```

### Associated Types vs Generic Parameters

```rust
// Generic — allows multiple implementations of the same trait for the same type
trait Add<Rhs> {
    fn add(self, rhs: Rhs) -> Self;
}

// Associated type — one implementation per type (cleaner)
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

---

## Supertraits

Require that a trait depends on another:

```rust
trait PrettyPrint: Display {
    fn pretty_print(&self) {
        println!("=== {} ===", self);  // can use Display methods
    }
}
```

Implementing `PrettyPrint` requires also implementing `Display`.

---

## The Newtype Pattern

Wrap a type to implement foreign traits or add behaviour:

```rust
struct Meters(f64);
struct Seconds(f64);

impl Display for Meters {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.2}m", self.0)
    }
}

// Prevents mixing up units — Meters and Seconds are different types
fn speed(distance: Meters, time: Seconds) -> f64 {
    distance.0 / time.0
}
```

---

## Key Takeaways

- Traits define shared behaviour. Types implement traits implicitly — no `implements` keyword.
- Generics with trait bounds enable type-safe polymorphism. The compiler monomorphises generic code for zero-cost performance.
- Use `impl Trait` for static dispatch (fast, most common) and `dyn Trait` for dynamic dispatch (heterogeneous collections, plugins).
- Standard traits (`Display`, `Debug`, `Clone`, `From`, `Iterator`) form the foundation of the ecosystem.
- Associated types simplify traits where each type has exactly one implementation.
- The newtype pattern wraps foreign types to implement traits or enforce type safety (unit types, domain types).
