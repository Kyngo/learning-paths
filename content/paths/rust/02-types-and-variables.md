---
title: "Types & Variables"
weight: 2
---

# Types & Variables

Rust is statically typed with powerful type inference. The compiler deduces most types, but you must be explicit in function signatures. Every value has exactly one owner, a fixed type, and a fixed size known at compile time (or stored behind a pointer).

---

## Scalar Types

| Type | Size | Range | Example |
|------|------|-------|---------|
| `i8` | 1 byte | -128 to 127 | `let x: i8 = -42;` |
| `i16` | 2 bytes | -32,768 to 32,767 | |
| `i32` | 4 bytes | -2³¹ to 2³¹-1 | Default integer type |
| `i64` | 8 bytes | -2⁶³ to 2⁶³-1 | |
| `i128` | 16 bytes | Very large | |
| `isize` | Pointer-sized | Platform-dependent | Index into collections |
| `u8` | 1 byte | 0 to 255 | Bytes, ASCII |
| `u16` | 2 bytes | 0 to 65,535 | |
| `u32` | 4 bytes | 0 to 2³²-1 | |
| `u64` | 8 bytes | 0 to 2⁶⁴-1 | |
| `u128` | 16 bytes | Very large | |
| `usize` | Pointer-sized | Platform-dependent | Array indices, sizes |
| `f32` | 4 bytes | ~7 digits | |
| `f64` | 8 bytes | ~15 digits | Default float type |
| `bool` | 1 byte | `true` / `false` | |
| `char` | 4 bytes | Unicode scalar value | `'a'`, `'🦀'` |

### Integer Literals

```rust
let decimal = 98_222;       // underscores for readability
let hex = 0xff;
let octal = 0o77;
let binary = 0b1111_0000;
let byte_literal = b'A';    // u8 only
```

### Integer Overflow

In debug mode, overflow **panics**. In release mode, it wraps (two's complement). To be explicit:

```rust
let x: u8 = 255;
x.wrapping_add(1)   // 0
x.checked_add(1)    // None
x.saturating_add(1) // 255
x.overflowing_add(1) // (0, true)
```

---

## Compound Types

### Tuples

Fixed-size, heterogeneous:

```rust
let tup: (i32, f64, bool) = (500, 6.4, true);

// Destructuring
let (x, y, z) = tup;

// Index access
let first = tup.0;
let second = tup.1;

// Unit tuple (no values) — used as "void"
let unit: () = ();
```

### Arrays

Fixed-size, homogeneous, stack-allocated:

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0; 10];  // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

let first = a[0];
let len = a.len();

// Out-of-bounds access panics at runtime (bounds-checked)
// let bad = a[10];  // panic: index out of bounds
```

---

## Strings

Rust has two primary string types:

| Type | Stored | Mutability | Ownership | Use |
|------|--------|-----------|-----------|-----|
| `&str` | Stack/static | Immutable | Borrowed | String slices, literals |
| `String` | Heap | Growable | Owned | Building, modifying strings |

```rust
// String literal — type is &str (borrowed, static lifetime)
let s1: &str = "hello";

// Owned String — heap-allocated
let s2: String = String::from("hello");
let s3: String = "hello".to_string();

// Convert
let borrowed: &str = &s2;           // String → &str (cheap, just a reference)
let owned: String = s1.to_string(); // &str → String (allocates)
```

### String Operations

```rust
let mut s = String::from("Hello");
s.push(' ');             // append char
s.push_str("World");    // append &str
s += "!";               // equivalent to push_str

let combined = format!("{} {}", "Hello", "World");
let len = s.len();       // byte length (not char count)
let chars = s.chars().count();  // character count

// Slicing (by byte index — must be at char boundaries)
let hello = &s[0..5];

// Iteration
for c in s.chars() { ... }  // by character
for b in s.bytes() { ... }  // by byte
```

**Strings are UTF-8.** Indexing by position (`s[0]`) is not allowed because a character can be multiple bytes. Use `.chars()` or `.bytes()`.

---

## Type Inference

The compiler infers types from context:

```rust
let x = 42;           // i32 (default integer)
let y = 3.14;         // f64 (default float)
let z = true;         // bool
let name = "Alice";   // &str

// Sometimes you need to help
let parsed: i64 = "42".parse().unwrap();
let nums: Vec<i32> = vec![1, 2, 3];
```

### Type Aliases

```rust
type Meters = f64;
type Result<T> = std::result::Result<T, Box<dyn std::error::Error>>;
```

---

## Constants and Statics

```rust
// Constant — inlined at every use site, no memory address
const MAX_RETRIES: u32 = 3;
const PI: f64 = 3.14159265358979;

// Static — has a fixed memory address, lives for the entire program
static GREETING: &str = "Hello";

// Mutable static — requires unsafe (global mutable state)
static mut COUNTER: u32 = 0;
```

| | `const` | `static` |
|-|---------|----------|
| Memory | Inlined (no address) | Fixed address |
| Mutability | Always immutable | Can be `mut` (requires `unsafe`) |
| Lifetime | Evaluated at compile time | `'static` |
| Use | Mathematical constants, config values | Global singletons, FFI interop |

---

## Type Conversions

Rust has **no implicit conversions** — every conversion is explicit:

```rust
let x: i32 = 42;
let y: f64 = x as f64;       // numeric cast
let z: u8 = x as u8;         // truncating cast (if x > 255)

// Fallible conversion via TryFrom
let big: i32 = 300;
let small: u8 = u8::try_from(big)?;  // returns Err if out of range

// From/Into traits
let s: String = String::from("hello");
let s: String = "hello".into();       // equivalent (Into is auto-derived from From)
```

### The `From` / `Into` Traits

```rust
// If you implement From, you get Into for free
impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 1.8 + 32.0)
    }
}

let f: Fahrenheit = Celsius(100.0).into();
```

---

## Expressions vs Statements

Almost everything in Rust is an expression (returns a value):

```rust
// if is an expression
let x = if condition { 5 } else { 10 };

// Block is an expression (last expression without ; is the return value)
let y = {
    let a = 1;
    let b = 2;
    a + b  // no semicolon = this is the return value
};
// y == 3

// Adding a semicolon turns an expression into a statement (returns ())
let z = {
    let a = 1;
    a + 1;  // semicolon = this is discarded, block returns ()
};
// z == ()
```

This is why Rust functions don't need `return` for the last expression:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b  // no semicolon, no return keyword
}
```

---

## Key Takeaways

- Rust has no implicit type conversions. Use `as` for primitive casts, `From`/`Into` for semantic conversions, `TryFrom` for fallible ones.
- Two string types: `&str` (borrowed slice, lightweight) and `String` (owned, heap-allocated, growable).
- Variables are immutable by default. Overflow panics in debug, wraps in release — use `checked_add`/`saturating_add` for explicit control.
- Almost everything is an expression. `if`, blocks, and `match` all return values.
- The last expression in a block (without `;`) is its return value. This is idiomatic — don't use `return` unless exiting early.
- `char` is 4 bytes (Unicode scalar value), not 1 byte. Strings are UTF-8 byte sequences.
