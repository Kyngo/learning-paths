---
title: "Getting Started"
weight: 1
---

# Getting Started

Rust's toolchain is self-contained and well-designed. `rustup` manages versions, `cargo` handles building, testing, and dependency management. Within minutes of installation you can compile, run, test, and lint Rust code.

---

## Installation

```bash
# Install rustup (manages Rust versions)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Verify
rustc --version    # compiler
cargo --version    # build tool + package manager
rustup --version   # toolchain manager
```

### Toolchain Management

```bash
rustup update                    # update to latest stable
rustup default stable            # use stable channel
rustup default nightly           # use nightly (for bleeding-edge features)
rustup component add clippy      # add linter
rustup component add rustfmt     # add formatter
rustup target add wasm32-unknown-unknown  # add WebAssembly target
```

---

## Hello World

```rust
fn main() {
    println!("Hello, World!");
}
```

```bash
rustc main.rs       # compile directly
./main              # run

# But in practice, always use cargo:
cargo new hello     # creates a project
cd hello
cargo run           # compile + run
```

### What's `println!`?

The `!` indicates a **macro**, not a function. Macros are expanded at compile time — `println!` accepts variable argument counts and format strings that are checked at compile time.

```rust
let name = "Alice";
let age = 30;
println!("Name: {}, Age: {}", name, age);     // positional
println!("Name: {name}, Age: {age}");          // inline (Rust 1.58+)
println!("{:?}", vec![1, 2, 3]);               // debug format
println!("{:#?}", complex_struct);              // pretty debug
```

---

## Cargo

Cargo is Rust's build tool and package manager — equivalent to `go` + `npm` combined.

### Creating a Project

```bash
cargo new myproject       # binary (executable)
cargo new mylib --lib     # library
```

### Project Structure

```
myproject/
├── Cargo.toml            # manifest (dependencies, metadata)
├── Cargo.lock            # exact dependency versions (commit this for binaries)
├── src/
│   ├── main.rs           # binary entry point
│   └── lib.rs            # library root (if library)
├── tests/                # integration tests
│   └── integration_test.rs
├── benches/              # benchmarks
│   └── bench.rs
└── examples/             # example programs
    └── demo.rs
```

### Cargo.toml

```toml
[package]
name = "myproject"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
anyhow = "1.0"

[dev-dependencies]
assert_cmd = "2.0"
tempfile = "3.0"

[profile.release]
opt-level = 3
lto = true          # link-time optimization
strip = true        # strip debug symbols
```

### Essential Cargo Commands

```bash
cargo build                # compile (debug mode)
cargo build --release      # compile (optimized)
cargo run                  # compile + run
cargo run -- arg1 arg2     # pass args to the binary
cargo test                 # run all tests
cargo test test_name       # run matching tests
cargo bench                # run benchmarks
cargo check                # type-check without building (fast)
cargo clippy               # lint (better than compiler warnings)
cargo fmt                  # format code
cargo doc --open           # generate and view documentation
cargo add serde            # add a dependency (Rust 1.62+)
cargo update               # update dependencies
cargo tree                 # show dependency tree
```

### `cargo check` vs `cargo build`

`cargo check` runs the compiler but stops before code generation. It is 2-5× faster than a full build. Use it during development for rapid type-checking feedback.

---

## Variables and Mutability

Variables in Rust are **immutable by default**:

```rust
let x = 5;
x = 6;  // ERROR: cannot assign twice to immutable variable

let mut y = 5;
y = 6;  // OK — explicitly marked mutable
```

This is the opposite of most languages and is intentional. Immutability by default makes code easier to reason about and enables safe concurrency.

### Shadowing

You can redeclare a variable with the same name:

```rust
let x = 5;
let x = x + 1;      // new binding, shadows the old one
let x = x * 2;      // 12

// Shadowing can change the type
let spaces = "   ";
let spaces = spaces.len();  // now an integer
```

Shadowing is different from `mut` — it creates a new variable, which can have a different type.

---

## Comments

```rust
// Line comment

/* Block comment */

/// Documentation comment (generates HTML docs)
/// Supports Markdown formatting.
///
/// # Examples
///
/// ```
/// let result = add(2, 3);
/// assert_eq!(result, 5);
/// ```
fn add(a: i32, b: i32) -> i32 {
    a + b
}

//! Module-level documentation (placed at the top of a file)
//! Describes the purpose of the module or crate.
```

Documentation comments with `///` are tested by `cargo test` — the code in `# Examples` blocks is compiled and run.

---

## Printing and Debugging

```rust
// Standard output
println!("value: {}", x);         // Display trait
println!("debug: {:?}", x);       // Debug trait
println!("pretty: {:#?}", x);     // Debug with indentation
print!("no newline");             // without newline
eprintln!("error: {}", msg);      // to stderr

// Debug during development
dbg!(x);           // prints file:line, expression, and value to stderr
dbg!(&my_vec);     // returns the value, so you can chain: dbg!(x + 1)
```

### Deriving Debug

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
}

let u = User { name: "Alice".into(), age: 30 };
println!("{:?}", u);   // User { name: "Alice", age: 30 }
println!("{:#?}", u);  // pretty-printed
```

---

## Cargo Clippy (Linting)

```bash
cargo clippy           # run linter
cargo clippy -- -W clippy::pedantic  # stricter checks
```

Clippy catches hundreds of patterns: unnecessary clones, redundant closures, inefficient iterations, non-idiomatic code. It is more useful than `rustc` warnings alone.

### Configuration (clippy.toml or Cargo.toml)

```toml
# In Cargo.toml
[lints.clippy]
pedantic = "warn"
nursery = "warn"
unwrap_used = "warn"
```

---

## Key Takeaways

- Install via `rustup`, build via `cargo`. Never use `rustc` directly for real projects.
- `cargo check` is your inner-loop tool — faster than `cargo build`, catches all type errors.
- Variables are immutable by default. Use `mut` only when you need to modify.
- Shadowing lets you reuse names (and even change types) without mutability.
- `println!` is a macro, not a function. The `!` is the giveaway. Format strings are checked at compile time.
- `cargo clippy` is non-negotiable. It catches more issues than the compiler's built-in warnings.
- Documentation comments (`///`) with code examples are tested by `cargo test` — your docs stay correct.
