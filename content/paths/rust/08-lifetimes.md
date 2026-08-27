---
title: "Lifetimes"
weight: 8
---

# Lifetimes

Lifetimes are Rust's way of ensuring that references are always valid. They are annotations that tell the compiler how long a reference must live. Most of the time, the compiler infers lifetimes automatically. You only need explicit annotations when the compiler cannot figure it out.

---

## What Lifetimes Prevent

```c
// C — dangling pointer (compiles, crashes at runtime)
int* dangling() {
    int x = 42;
    return &x;  // x is freed when function returns — pointer is invalid
}
```

```rust
// Rust — compiler rejects this
fn dangling() -> &i32 {
    let x = 42;
    &x  // ERROR: `x` does not live long enough
}
```

The compiler tracks how long every reference is valid and rejects code where a reference outlives its referent.

---

## Lifetime Annotations

Lifetime annotations do not change how long values live — they describe **relationships** between reference lifetimes so the compiler can verify correctness.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

Read `'a` as "some lifetime." The signature says: "the returned reference lives as long as the shorter of x and y's lifetimes."

### Why Is This Needed?

Without the annotation, the compiler cannot know which input the return value refers to:

```rust
let result;
{
    let s1 = String::from("long");
    let s2 = String::from("short");
    result = longest(&s1, &s2);  // which one does result point to?
}
// If result points to s2, and s2 is dropped here, result is dangling
```

With the annotation `'a`, the compiler knows `result` must not outlive either input.

---

## Lifetime Elision Rules

The compiler infers lifetimes in common cases. You do not need annotations when the rules apply:

**Rule 1:** Each input reference gets its own lifetime parameter.

```rust
fn first(s: &str) -> &str { ... }
// becomes: fn first<'a>(s: &'a str) -> &'a str
```

**Rule 2:** If there is exactly one input lifetime, it is assigned to all output lifetimes.

```rust
fn first_word(s: &str) -> &str { ... }
// Only one input ref → output gets the same lifetime
```

**Rule 3:** If one of the parameters is `&self` or `&mut self`, the lifetime of self is assigned to all output lifetimes.

```rust
impl User {
    fn name(&self) -> &str { &self.name }
    // self's lifetime → output lifetime
}
```

### When You NEED Explicit Lifetimes

- Multiple input references and the output borrows from them
- Structs that hold references
- Trait implementations with references

---

## Structs with References

If a struct holds a reference, it needs a lifetime parameter:

```rust
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    fn new(text: &'a str) -> Self {
        Excerpt { text }
    }

    fn first_sentence(&self) -> &str {
        self.text.split('.').next().unwrap_or(self.text)
    }
}

let novel = String::from("Call me Ishmael. Some years ago...");
let excerpt = Excerpt::new(&novel.split('.').next().unwrap());
```

The `'a` annotation tells the compiler: "an `Excerpt` cannot outlive the string it references."

### The Alternative: Own the Data

```rust
// Instead of a reference with a lifetime...
struct Excerpt<'a> {
    text: &'a str,  // borrows — complex lifetime management
}

// ...own the data
struct Excerpt {
    text: String,  // owns — simpler, no lifetime annotation
}
```

**Rule of thumb:** If the lifetime annotations are making your code complex, consider owning the data instead. Clone if necessary — correctness first, optimise later.

---

## `'static` Lifetime

`'static` means the reference is valid for the entire program duration:

```rust
// String literals have 'static lifetime (embedded in binary)
let s: &'static str = "hello";

// Constants
static CONFIG: &str = "production";
```

### `'static` in Trait Bounds

`T: 'static` does not mean "a static reference." It means "T contains no non-`'static` references" — i.e., T either owns its data or contains only `'static` references.

```rust
fn spawn_task(task: impl FnOnce() + Send + 'static) {
    std::thread::spawn(task);
}
// The closure must own all its data (or have 'static refs)
// because the thread may outlive the caller
```

---

## Multiple Lifetimes

When different references have different lifetimes:

```rust
fn first_or_default<'a, 'b>(first: &'a str, default: &'b str) -> &'a str {
    if first.is_empty() {
        // Cannot return default here — it has lifetime 'b, not 'a
        panic!("first is empty")
    }
    first
}
```

If the function needs to return either input:

```rust
fn first_or_default<'a>(first: &'a str, default: &'a str) -> &'a str {
    if first.is_empty() { default } else { first }
}
// Both inputs share lifetime 'a — caller must ensure both live long enough
```

---

## Lifetime Bounds on Generics

```rust
// T must outlive lifetime 'a
fn print_ref<'a, T: Display + 'a>(reference: &'a T) {
    println!("{}", reference);
}

// In struct definitions
struct Wrapper<'a, T: 'a> {
    value: &'a T,
}
```

---

## Common Lifetime Patterns

### Return a Reference to Input

```rust
fn first_word(s: &str) -> &str {
    // Elision rule 2: output lifetime = input lifetime
    s.split_whitespace().next().unwrap_or("")
}
```

### Return a Reference from a Struct Method

```rust
impl Config {
    fn get(&self, key: &str) -> Option<&str> {
        // Elision rule 3: output lifetime = self's lifetime
        self.values.get(key).map(|v| v.as_str())
    }
}
```

### When to Clone Instead of Borrow

| Borrow (Reference) | Clone (Own) |
|-------------------|-------------|
| Performance-critical hot path | Clarity matters more than performance |
| Data is large | Data is small (String, small struct) |
| Caller manages the data | Callee needs independent ownership |
| Lifetime is straightforward | Lifetime annotations are getting complex |

---

## Key Takeaways

- Lifetimes ensure references are always valid. They describe relationships, not durations.
- The compiler infers lifetimes in most cases (elision rules). You only annotate when it cannot.
- `'a` on a function signature means "the output reference lives as long as the shortest-lived input."
- Structs that hold references need lifetime parameters. Consider owning the data instead for simplicity.
- `'static` means "valid for the entire program" for references, or "contains no short-lived references" for types.
- When lifetime annotations make code complex, own the data. Correctness and clarity beat micro-optimisation.
