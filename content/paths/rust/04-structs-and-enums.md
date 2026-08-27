---
title: "Structs & Enums"
weight: 4
---

# Structs & Enums

Structs group related data. Enums represent values that can be one of several variants. Together with pattern matching, they form Rust's primary data modelling toolkit — replacing classes, inheritance, and null in other languages.

---

## Structs

```rust
struct User {
    id: u64,
    name: String,
    email: String,
    active: bool,
}

let user = User {
    id: 1,
    name: String::from("Alice"),
    email: String::from("alice@example.com"),
    active: true,
};

println!("{}", user.name);
```

### Field Init Shorthand

```rust
fn new_user(name: String, email: String) -> User {
    User {
        id: generate_id(),
        name,     // shorthand: field name matches variable name
        email,    // same
        active: true,
    }
}
```

### Struct Update Syntax

```rust
let user2 = User {
    email: String::from("bob@example.com"),
    ..user  // take remaining fields from user (MOVES String fields)
};
```

### Tuple Structs

```rust
struct Color(u8, u8, u8);
struct Point(f64, f64);

let red = Color(255, 0, 0);
let origin = Point(0.0, 0.0);
println!("r={}", red.0);
```

### Unit Structs

```rust
struct AlwaysEqual;  // no fields, used as markers or trait implementations
```

---

## Methods

Defined in `impl` blocks:

```rust
impl User {
    // Associated function (constructor) — no self
    fn new(name: String, email: String) -> Self {
        Self {
            id: generate_id(),
            name,
            email,
            active: true,
        }
    }

    // Method — borrows self immutably
    fn full_info(&self) -> String {
        format!("{} <{}>", self.name, self.email)
    }

    // Method — borrows self mutably
    fn deactivate(&mut self) {
        self.active = false;
    }

    // Method — consumes self (takes ownership)
    fn into_name(self) -> String {
        self.name
    }
}

let mut user = User::new("Alice".into(), "alice@example.com".into());
println!("{}", user.full_info());
user.deactivate();
let name = user.into_name();  // user is consumed, no longer usable
```

### Receiver Types

| Receiver | Meaning | When |
|----------|---------|------|
| `&self` | Immutable borrow | Read-only access |
| `&mut self` | Mutable borrow | Modify the struct |
| `self` | Take ownership | Transform or consume the struct |

---

## Enums

Enums in Rust are **algebraic data types** (tagged unions). Each variant can hold different data:

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);
let loopback = IpAddr::V6(String::from("::1"));
```

### Enums with Named Fields

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}
```

### Methods on Enums

```rust
impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Triangle { base, height } => 0.5 * base * height,
        }
    }
}
```

---

## Pattern Matching

`match` is exhaustive — you must handle every variant:

```rust
fn describe(shape: &Shape) -> String {
    match shape {
        Shape::Circle { radius } => format!("circle with radius {radius}"),
        Shape::Rectangle { width, height } => format!("{width}x{height} rectangle"),
        Shape::Triangle { .. } => "triangle".to_string(),
    }
}
```

### Match Arms

```rust
match value {
    1 => println!("one"),
    2 | 3 => println!("two or three"),
    4..=10 => println!("four to ten"),
    n if n < 0 => println!("negative: {n}"),  // guard
    _ => println!("something else"),           // wildcard
}
```

### `if let` and `let else`

```rust
// if let — handle one variant, ignore the rest
if let Some(value) = optional {
    println!("got {value}");
}

// let else (Rust 1.65+) — handle the happy path, diverge on mismatch
let Some(value) = optional else {
    return Err(anyhow!("missing value"));
};
// value is available here
```

### `while let`

```rust
while let Some(item) = stack.pop() {
    process(item);
}
```

---

## `Option<T>` — Null Replacement

Rust has no null. Instead, `Option<T>` represents a value that might be absent:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

```rust
fn find_user(id: u64) -> Option<User> {
    if id == 1 {
        Some(User { id: 1, name: "Alice".into(), /* ... */ })
    } else {
        None
    }
}

match find_user(1) {
    Some(user) => println!("found: {}", user.name),
    None => println!("not found"),
}

// Combinators
let name = find_user(1).map(|u| u.name).unwrap_or("unknown".into());
let user = find_user(1).ok_or(anyhow!("user not found"))?;
```

### Common Option Methods

| Method | Behaviour |
|--------|-----------|
| `unwrap()` | Returns value or panics |
| `expect("msg")` | Returns value or panics with message |
| `unwrap_or(default)` | Returns value or default |
| `unwrap_or_else(\|\| expr)` | Returns value or computes default |
| `map(f)` | Transforms the inner value |
| `and_then(f)` | Chains operations that return Option |
| `ok_or(err)` | Converts Option to Result |
| `is_some()` / `is_none()` | Boolean check |

---

## `Result<T, E>` — Error Handling

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

```rust
fn parse_port(s: &str) -> Result<u16, String> {
    s.parse::<u16>().map_err(|e| format!("invalid port: {e}"))
}

match parse_port("8080") {
    Ok(port) => println!("port: {port}"),
    Err(e) => eprintln!("error: {e}"),
}
```

Result is covered in depth in the Error Handling section.

---

## Deriving Traits

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct User {
    id: u64,
    name: String,
}

#[derive(Debug, Clone, PartialEq)]
enum Status {
    Active,
    Inactive,
    Suspended,
}
```

| Derive | Provides |
|--------|---------|
| `Debug` | `{:?}` formatting |
| `Clone` | `.clone()` method |
| `Copy` | Implicit copy (stack-only types) |
| `PartialEq` | `==` and `!=` |
| `Eq` | Total equality (PartialEq + reflexive) |
| `Hash` | Hashable (for HashMap/HashSet keys) |
| `PartialOrd` | `<`, `>`, `<=`, `>=` |
| `Ord` | Total ordering |
| `Default` | `Default::default()` |
| `Serialize` / `Deserialize` | serde serialisation (external crate) |

---

## Key Takeaways

- Structs hold related data; enums represent one-of-many variants. Together they replace classes and inheritance.
- Enums can hold data in each variant — they are algebraic data types, not just integer constants.
- `match` is exhaustive — the compiler forces you to handle every case. This eliminates missed branches.
- `Option<T>` replaces null. You cannot use a value that might be absent without checking first.
- `Result<T, E>` replaces exceptions. Errors are values in the return type, not hidden control flow.
- `if let` and `let else` simplify matching when you only care about one variant.
- Derive macros (`Debug`, `Clone`, `PartialEq`) generate boilerplate trait implementations automatically.
