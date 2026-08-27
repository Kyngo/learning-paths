---
title: "Error Handling"
weight: 5
---

# Error Handling

Rust uses `Result<T, E>` for recoverable errors and `panic!` for unrecoverable ones. There are no exceptions, no try/catch — errors are values in the type system, and the compiler forces you to handle them.

---

## Result<T, E>

```rust
enum Result<T, E> {
    Ok(T),    // success with value
    Err(E),   // failure with error
}
```

```rust
use std::fs;

fn read_config(path: &str) -> Result<String, std::io::Error> {
    fs::read_to_string(path)
}

match read_config("config.toml") {
    Ok(contents) => println!("{contents}"),
    Err(e) => eprintln!("Failed to read config: {e}"),
}
```

---

## The `?` Operator

`?` propagates errors — if the Result is Err, return it immediately; if Ok, unwrap the value:

```rust
fn read_username() -> Result<String, std::io::Error> {
    let contents = fs::read_to_string("username.txt")?;  // returns Err if failed
    Ok(contents.trim().to_string())
}
```

`?` can be chained:

```rust
fn read_config() -> Result<Config, Box<dyn Error>> {
    let data = fs::read_to_string("config.toml")?;
    let config: Config = toml::from_str(&data)?;
    Ok(config)
}
```

### `?` Does Automatic Conversion

If the error type implements `From`, `?` converts automatically:

```rust
// std::io::Error → Box<dyn Error> via From
// toml::de::Error → Box<dyn Error> via From
// Both work with ? in the same function
```

---

## Error Types

### String Errors (Quick and Dirty)

```rust
fn validate(name: &str) -> Result<(), String> {
    if name.is_empty() {
        return Err("name cannot be empty".to_string());
    }
    Ok(())
}
```

### Box<dyn Error> (Catch-All)

```rust
use std::error::Error;

fn run() -> Result<(), Box<dyn Error>> {
    let data = fs::read_to_string("data.json")?;
    let parsed: Value = serde_json::from_str(&data)?;
    Ok(())
}
```

### Custom Error Types

```rust
#[derive(Debug)]
enum AppError {
    NotFound(String),
    Unauthorized,
    Database(sqlx::Error),
    Validation(String),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AppError::NotFound(id) => write!(f, "resource not found: {id}"),
            AppError::Unauthorized => write!(f, "unauthorized"),
            AppError::Database(e) => write!(f, "database error: {e}"),
            AppError::Validation(msg) => write!(f, "validation: {msg}"),
        }
    }
}

impl std::error::Error for AppError {}

impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        AppError::Database(e)
    }
}
```

---

## `thiserror` and `anyhow`

### `thiserror` — For Libraries (Structured Errors)

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("resource not found: {0}")]
    NotFound(String),

    #[error("unauthorized")]
    Unauthorized,

    #[error("database error")]
    Database(#[from] sqlx::Error),

    #[error("validation: {0}")]
    Validation(String),
}
```

`thiserror` generates `Display`, `Error`, and `From` implementations automatically.

### `anyhow` — For Applications (Convenience)

```rust
use anyhow::{Context, Result, bail, ensure};

fn process() -> Result<()> {
    let config = fs::read_to_string("config.toml")
        .context("failed to read config file")?;

    ensure!(!config.is_empty(), "config file is empty");

    if invalid_state {
        bail!("unexpected state: {state}");
    }

    Ok(())
}
```

| Crate | Use | Error Type |
|-------|-----|-----------|
| `thiserror` | Libraries — callers need to match on error variants | Your custom enum |
| `anyhow` | Applications — you just need context and propagation | `anyhow::Error` (type-erased) |

---

## Panic

`panic!` is for unrecoverable errors — bugs, impossible states, violated invariants:

```rust
panic!("this should never happen");

// Common panicking operations:
vec[100]       // index out of bounds
unwrap()       // called on None or Err
expect("msg")  // same as unwrap with a message
```

### When to Panic vs Return Error

| Panic | Result |
|-------|--------|
| Bug in your code | Expected failure |
| Impossible state | I/O error, network error |
| Test assertions | User input validation |
| Prototype / quick script | Production service |
| `unwrap()` in tests | `?` in production |

---

## Common Patterns

### Map and Combine

```rust
// Transform Ok value
let len = fs::read_to_string("file.txt").map(|s| s.len());

// Chain fallible operations
let user = get_id()
    .and_then(|id| find_user(id))
    .and_then(|user| validate(user));

// Provide default on error
let port = env::var("PORT")
    .unwrap_or_else(|_| "8080".to_string())
    .parse::<u16>()
    .unwrap_or(8080);
```

### Collecting Results

```rust
// Fail on first error
let numbers: Result<Vec<i32>, _> = strings.iter()
    .map(|s| s.parse::<i32>())
    .collect();

// Partition successes and failures
let (ok, err): (Vec<_>, Vec<_>) = results.into_iter().partition(Result::is_ok);
```

---

## Key Takeaways

- `Result<T, E>` is the standard error type. `?` propagates errors concisely.
- Use `thiserror` for library error types (structured, matchable). Use `anyhow` for application code (context, convenience).
- `?` auto-converts errors via `From`. Implement `From` on your error types to enable `?` across type boundaries.
- `panic!` is for bugs. `Result` is for expected failures. Calling `unwrap()` in production code is a code smell.
- Add context with `.context("msg")?` (anyhow) or `map_err` to make error messages useful for debugging.
