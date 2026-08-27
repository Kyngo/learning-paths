---
title: "Modules & Cargo"
weight: 11
---

# Modules & Cargo

Rust's module system organises code within a crate. Cargo manages building, testing, dependencies, workspaces, and publishing. Together they handle projects from single files to large monorepos.

---

## Module System

### Declaring Modules

```rust
// src/lib.rs or src/main.rs
mod handler;     // looks for src/handler.rs or src/handler/mod.rs
mod service;     // looks for src/service.rs or src/service/mod.rs

pub mod api {    // inline module
    pub fn health() -> &'static str { "ok" }
}
```

### File Layout

```
src/
├── main.rs           // crate root for binary
├── lib.rs            // crate root for library
├── handler.rs        // module: crate::handler
├── handler/
│   ├── mod.rs        // alternative to handler.rs (older style)
│   └── middleware.rs  // submodule: crate::handler::middleware
├── service.rs        // module: crate::service
└── model/
    ├── mod.rs        // module: crate::model
    ├── user.rs       // submodule: crate::model::user
    └── order.rs      // submodule: crate::model::order
```

### Visibility

```rust
pub struct User {          // public struct
    pub name: String,      // public field
    email: String,         // private field (default)
}

pub fn create_user() { }  // public function
fn validate() { }         // private function (default)

pub(crate) fn internal() { }  // visible within the crate only
pub(super) fn parent() { }   // visible to parent module only
```

### Use and Re-exports

```rust
// Import
use crate::model::user::User;
use crate::service::{UserService, OrderService};
use std::collections::HashMap;

// Glob import (avoid in production code)
use std::io::*;

// Re-export — make internal items available at a higher level
pub use crate::model::user::User;
// Now consumers can use: your_crate::User instead of your_crate::model::user::User
```

---

## Crates

A **crate** is a compilation unit — the smallest amount of code the compiler considers at once.

| Crate Type | Purpose | Entry Point |
|-----------|---------|-------------|
| Binary | Executable | `src/main.rs` |
| Library | Reusable code | `src/lib.rs` |

A package (Cargo.toml) can contain one library crate and multiple binary crates.

### Multiple Binaries

```
src/
├── main.rs           // default binary
├── lib.rs            // library
└── bin/
    ├── server.rs     // cargo run --bin server
    └── cli.rs        // cargo run --bin cli
```

---

## Cargo Features

Conditional compilation — enable or disable parts of your crate:

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]
xml = ["dep:quick-xml"]
full = ["json", "xml"]

[dependencies]
serde = "1.0"
serde_json = { version = "1.0", optional = true }
quick-xml = { version = "0.31", optional = true }
```

```rust
#[cfg(feature = "json")]
pub fn to_json(value: &impl Serialize) -> String {
    serde_json::to_string(value).unwrap()
}
```

```bash
cargo build                        # default features (json)
cargo build --features xml         # default + xml
cargo build --all-features         # everything
cargo build --no-default-features  # nothing
```

---

## Workspaces

Monorepo with multiple crates sharing dependencies and a single `Cargo.lock`:

```toml
# Root Cargo.toml
[workspace]
members = [
    "crates/core",
    "crates/api",
    "crates/cli",
]

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

```toml
# crates/api/Cargo.toml
[package]
name = "api"

[dependencies]
core = { path = "../core" }
serde.workspace = true    # use workspace version
tokio.workspace = true
```

```bash
cargo build -p api        # build specific crate
cargo test --workspace    # test all crates
```

---

## Testing

### Unit Tests (Same File)

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_add_negative() {
        assert_eq!(add(-1, 1), 0);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn test_overflow() {
        add(i32::MAX, 1);
    }

    #[test]
    fn test_with_result() -> Result<(), String> {
        if add(1, 1) == 2 {
            Ok(())
        } else {
            Err("math is broken".to_string())
        }
    }
}
```

### Integration Tests (Separate Files)

```rust
// tests/integration_test.rs
use mylib::add;

#[test]
fn it_adds() {
    assert_eq!(add(2, 2), 4);
}
```

Integration tests are in the `tests/` directory, each file is a separate crate, and they can only use public APIs.

### Doc Tests

```rust
/// Adds two numbers.
///
/// # Examples
///
/// ```
/// assert_eq!(mylib::add(2, 3), 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

`cargo test` runs doc examples as tests.

### Running Tests

```bash
cargo test                      # all tests
cargo test test_name            # filter by name
cargo test -- --nocapture       # show println output
cargo test -- --ignored         # run ignored tests
cargo test -p crate_name        # specific crate
cargo test --doc                # doc tests only
```

---

## Benchmarks

### Criterion (Community Standard)

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_benchmark"
harness = false
```

```rust
// benches/my_benchmark.rs
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_sort(c: &mut Criterion) {
    let data: Vec<i32> = (0..1000).rev().collect();
    c.bench_function("sort 1000", |b| {
        b.iter(|| {
            let mut d = data.clone();
            d.sort();
        })
    });
}

criterion_group!(benches, bench_sort);
criterion_main!(benches);
```

```bash
cargo bench
```

---

## Publishing to crates.io

```bash
cargo login                 # authenticate with API token
cargo package                # create .crate file
cargo publish                # upload to crates.io
cargo publish --dry-run      # verify without uploading
```

### Cargo.toml for Publishing

```toml
[package]
name = "my-crate"
version = "0.1.0"
edition = "2021"
description = "A brief description"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/repo"
documentation = "https://docs.rs/my-crate"
readme = "README.md"
keywords = ["keyword1", "keyword2"]
categories = ["category"]
```

---

## Key Takeaways

- Modules organise code. Visibility is private by default — use `pub` to expose items.
- A crate is a compilation unit. A package (Cargo.toml) contains crates. A workspace groups packages.
- Features enable conditional compilation — use them instead of build-time flags.
- Unit tests go in the same file (`#[cfg(test)]`). Integration tests go in `tests/`. Doc tests go in `///` comments.
- Workspaces share a single `Cargo.lock` and dependency resolution — use them for monorepos.
- `cargo publish` uploads to crates.io. Use `cargo publish --dry-run` to verify first.
