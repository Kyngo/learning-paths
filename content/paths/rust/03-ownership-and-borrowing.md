---
title: "Ownership & Borrowing"
weight: 3
---

# Ownership & Borrowing

This is the concept that makes Rust unique. The ownership system is a set of compile-time rules that ensure memory safety without a garbage collector. Every value has exactly one owner, and when the owner goes out of scope, the value is dropped (freed). Borrowing lets you temporarily access values without taking ownership.

---

## The Three Rules of Ownership

1. **Each value has exactly one owner.**
2. **When the owner goes out of scope, the value is dropped.**
3. **Ownership can be transferred (moved) but not duplicated (for non-Copy types).**

```rust
{
    let s = String::from("hello");  // s owns the String
    // s is valid here
}  // s goes out of scope — String is dropped, memory freed
```

No garbage collector. No manual `free()`. The compiler inserts drop calls at the right places.

---

## Move Semantics

When you assign or pass a heap-allocated value, ownership **moves**:

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 is MOVED to s2

// println!("{}", s1);  // ERROR: s1 is no longer valid
println!("{}", s2);     // OK
```

### Why Move Instead of Copy?

A `String` is a struct on the stack containing a pointer, length, and capacity. The actual string data is on the heap. If we simply copied the struct, two variables would point to the same heap memory. When both go out of scope, the memory would be freed twice — a **double free** bug.

```
Stack:                  Heap:
s1: [ptr, 5, 5] ──→ ['h','e','l','l','o']
    ↑ MOVED
s2: [ptr, 5, 5] ──→ ['h','e','l','l','o']

After move, s1 is invalidated. Only s2 can access the data.
```

### Copy Types

Small, stack-only types implement the `Copy` trait — they are copied instead of moved:

```rust
let x = 42;
let y = x;     // x is COPIED (integers implement Copy)
println!("{x} {y}");  // both valid

// Copy types: all integers, floats, bool, char, tuples of Copy types
// NOT Copy: String, Vec, Box, any type with heap data
```

### Clone (Explicit Deep Copy)

```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // explicit deep copy
println!("{s1} {s2}"); // both valid
```

`clone()` is always explicit. You see it in the code and know it allocates.

---

## Functions and Ownership

Passing a value to a function moves it (for non-Copy types):

```rust
fn takes_ownership(s: String) {
    println!("{s}");
}  // s is dropped here

let name = String::from("Alice");
takes_ownership(name);
// println!("{name}");  // ERROR: name was moved
```

Returning a value from a function moves it to the caller:

```rust
fn gives_ownership() -> String {
    String::from("hello")  // moved to caller
}

let s = gives_ownership();  // s now owns the String
```

---

## Borrowing (References)

Borrowing lets you access a value without taking ownership. A reference is a pointer that is guaranteed to be valid.

### Immutable References (`&T`)

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
}  // s goes out of scope, but it doesn't own the data — nothing is dropped

let name = String::from("Alice");
let len = calculate_length(&name);  // borrow name
println!("{name} is {len} bytes");  // name is still valid
```

### Mutable References (`&mut T`)

```rust
fn append_greeting(s: &mut String) {
    s.push_str(", hello!");
}

let mut name = String::from("Alice");
append_greeting(&mut name);
println!("{name}");  // "Alice, hello!"
```

---

## The Borrowing Rules

1. **You can have either ONE mutable reference OR any number of immutable references — not both.**
2. **References must always be valid (no dangling references).**

```rust
let mut s = String::from("hello");

// Multiple immutable borrows — OK
let r1 = &s;
let r2 = &s;
println!("{r1} {r2}");

// Mutable borrow — OK (r1 and r2 are no longer used after this point)
let r3 = &mut s;
r3.push_str(" world");
println!("{r3}");

// CANNOT have mutable + immutable at the same time:
// let r4 = &s;
// let r5 = &mut s;
// println!("{r4} {r5}");  // ERROR
```

### Why This Restriction?

It prevents **data races** at compile time. A data race occurs when:
1. Two or more pointers access the same data simultaneously
2. At least one of them is writing
3. No synchronisation mechanism

The borrowing rules make condition 1+2 impossible: if someone is writing (`&mut`), nobody else can be reading (`&`).

---

## Slices

Slices are references to contiguous sequences — a **view** into owned data:

```rust
let s = String::from("hello world");

let hello: &str = &s[0..5];    // "hello"
let world: &str = &s[6..11];   // "world"
let full: &str = &s[..];       // "hello world"

// Array/Vec slices
let a = [1, 2, 3, 4, 5];
let slice: &[i32] = &a[1..4];  // [2, 3, 4]
```

**String literals are slices:** `"hello"` has type `&str` — a reference to string data embedded in the binary.

### Slices Prevent Dangling References

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[0..i];
        }
    }
    s
}

let mut s = String::from("hello world");
let word = first_word(&s);  // borrows s
// s.clear();  // ERROR: cannot mutate s while word borrows it
println!("{word}");
```

The compiler prevents `s.clear()` because `word` holds an immutable reference to `s`. In C, this would be a use-after-free.

---

## Ownership Patterns

### Return Ownership to Avoid Moves

```rust
fn process(s: String) -> String {
    // do something with s
    s  // return ownership back to caller
}
```

### Prefer Borrowing Over Moving

```rust
// BAD — takes ownership unnecessarily
fn print_name(name: String) {
    println!("{name}");
}

// GOOD — borrows, no ownership transfer
fn print_name(name: &str) {
    println!("{name}");
}
```

### The `&str` vs `&String` Rule

Accept `&str` in function parameters (not `&String`):

```rust
fn greet(name: &str) {
    println!("Hello, {name}!");
}

// Works with both:
greet("Alice");                        // &str literal
greet(&String::from("Bob"));           // &String auto-derefs to &str
```

---

## Drop and RAII

When a value goes out of scope, Rust calls its `drop` method (if implemented). This is **RAII (Resource Acquisition Is Initialization)** — the same pattern as C++ destructors.

```rust
{
    let file = File::open("data.txt")?;  // file opened
    // use file...
}  // file automatically closed here (Drop trait)

{
    let guard = mutex.lock()?;  // lock acquired
    // use shared data...
}  // lock automatically released here
```

You can implement `Drop` for custom cleanup:

```rust
struct Connection {
    addr: String,
}

impl Drop for Connection {
    fn drop(&mut self) {
        println!("Closing connection to {}", self.addr);
    }
}
```

---

## Key Takeaways

- Every value has exactly one owner. When the owner goes out of scope, the value is dropped.
- Assignment and function calls **move** heap-allocated values. Stack-only types (integers, bools) are **copied**.
- References borrow without taking ownership. Immutable (`&T`) or mutable (`&mut T`), never both simultaneously.
- The borrowing rules prevent data races and use-after-free at compile time. If it compiles, the memory is safe.
- Accept `&str` (not `&String`) in function parameters. Return `String` when the caller needs ownership.
- RAII (Drop) ensures resources are always cleaned up. No finally blocks, no defer, no GC — just scope.
- The borrow checker is strict but correct. When it rejects your code, there is a real potential bug. Learn to trust it.
