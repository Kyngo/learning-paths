---
title: "Patterns & Ecosystem"
weight: 12
---

# Patterns & Ecosystem

Rust has a rich ecosystem of crates and a strong culture of idiomatic patterns. This section covers the design patterns you will encounter in production Rust code and the key crates that form the practical foundation of the language.

---

## The Builder Pattern

For complex struct construction:

```rust
#[derive(Debug)]
struct Request {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout: std::time::Duration,
}

struct RequestBuilder {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout: std::time::Duration,
}

impl RequestBuilder {
    fn new(url: impl Into<String>) -> Self {
        Self {
            url: url.into(),
            method: "GET".to_string(),
            headers: Vec::new(),
            body: None,
            timeout: std::time::Duration::from_secs(30),
        }
    }

    fn method(mut self, method: impl Into<String>) -> Self {
        self.method = method.into();
        self
    }

    fn header(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.headers.push((key.into(), value.into()));
        self
    }

    fn body(mut self, body: impl Into<String>) -> Self {
        self.body = Some(body.into());
        self
    }

    fn timeout(mut self, timeout: std::time::Duration) -> Self {
        self.timeout = timeout;
        self
    }

    fn build(self) -> Request {
        Request {
            url: self.url,
            method: self.method,
            headers: self.headers,
            body: self.body,
            timeout: self.timeout,
        }
    }
}

// Usage
let req = RequestBuilder::new("https://api.example.com")
    .method("POST")
    .header("Content-Type", "application/json")
    .body(r#"{"key": "value"}"#)
    .build();
```

---

## The Newtype Pattern

Wrap a type for type safety, trait implementations, or semantic meaning:

```rust
struct UserId(u64);
struct OrderId(u64);

// These are different types — cannot be mixed
fn get_user(id: UserId) -> User { ... }
fn get_order(id: OrderId) -> Order { ... }

// get_user(OrderId(42));  // compile error!
```

### Deriving Traits for Newtypes

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);

impl Display for UserId {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "user:{}", self.0)
    }
}
```

---

## The Typestate Pattern

Use the type system to enforce state transitions at compile time:

```rust
struct Draft;
struct Published;

struct Article<State> {
    title: String,
    content: String,
    _state: std::marker::PhantomData<State>,
}

impl Article<Draft> {
    fn new(title: String) -> Self {
        Article {
            title,
            content: String::new(),
            _state: std::marker::PhantomData,
        }
    }

    fn edit(&mut self, content: String) {
        self.content = content;
    }

    fn publish(self) -> Article<Published> {
        Article {
            title: self.title,
            content: self.content,
            _state: std::marker::PhantomData,
        }
    }
}

impl Article<Published> {
    fn url(&self) -> String {
        format!("/articles/{}", self.title.to_lowercase().replace(' ', "-"))
    }
    // Cannot call edit() — wrong state
}

let mut draft = Article::<Draft>::new("Hello".to_string());
draft.edit("content".to_string());
let published = draft.publish();
println!("{}", published.url());
// published.edit(...);  // compile error — edit is not defined for Published
```

---

## The Extension Trait Pattern

Add methods to foreign types:

```rust
trait StringExt {
    fn truncate_with_ellipsis(&self, max_len: usize) -> String;
}

impl StringExt for str {
    fn truncate_with_ellipsis(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.to_string()
        } else {
            format!("{}…", &self[..max_len - 1])
        }
    }
}

"Hello, World!".truncate_with_ellipsis(8);  // "Hello, …"
```

---

## Key Ecosystem Crates

### Serialisation

| Crate | Purpose |
|-------|---------|
| `serde` | Serialisation framework (derive macros for struct ↔ any format) |
| `serde_json` | JSON |
| `serde_yaml` | YAML |
| `toml` | TOML (config files) |
| `csv` | CSV |

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    host: String,
    port: u16,
    #[serde(default)]
    debug: bool,
    #[serde(rename = "maxRetries")]
    max_retries: u32,
    #[serde(skip_serializing_if = "Option::is_none")]
    api_key: Option<String>,
}

// JSON
let json = serde_json::to_string_pretty(&config)?;
let config: Config = serde_json::from_str(&json)?;
```

### CLI

| Crate | Purpose |
|-------|---------|
| `clap` | Argument parsing (derive-based or builder) |
| `indicatif` | Progress bars |
| `dialoguer` | Interactive prompts |
| `colored` | Terminal colours |

```rust
use clap::Parser;

#[derive(Parser)]
#[command(name = "myapp", about = "Does useful things")]
struct Cli {
    /// Input file path
    #[arg(short, long)]
    input: String,

    /// Output format
    #[arg(short, long, default_value = "json")]
    format: String,

    /// Enable verbose output
    #[arg(short, long)]
    verbose: bool,
}

fn main() {
    let cli = Cli::parse();
    println!("Input: {}", cli.input);
}
```

### Async Runtime & HTTP

| Crate | Purpose |
|-------|---------|
| `tokio` | Async runtime |
| `reqwest` | HTTP client |
| `axum` | HTTP framework (by tokio team) |
| `actix-web` | HTTP framework (actor-based) |
| `tower` | Middleware/service abstractions |
| `tonic` | gRPC |

### Database

| Crate | Purpose |
|-------|---------|
| `sqlx` | Async SQL (compile-time query checking) |
| `diesel` | ORM / query builder |
| `sea-orm` | Async ORM |
| `rusqlite` | SQLite |

### Error Handling

| Crate | Purpose |
|-------|---------|
| `anyhow` | Application error handling (context, convenience) |
| `thiserror` | Library error types (derive Error) |
| `eyre` | anyhow alternative with custom reports |
| `color-eyre` | Pretty error reports with colour |

### Logging & Tracing

| Crate | Purpose |
|-------|---------|
| `tracing` | Structured logging and distributed tracing |
| `tracing-subscriber` | Configure tracing output |
| `log` | Simpler logging facade |
| `env_logger` | Configure log from RUST_LOG env var |

### Testing

| Crate | Purpose |
|-------|---------|
| `proptest` | Property-based testing |
| `rstest` | Test fixtures and parameterised tests |
| `mockall` | Mocking |
| `assert_cmd` | CLI integration testing |
| `wiremock` | HTTP mocking |
| `testcontainers` | Docker containers for tests |

---

## Project Template

```
myservice/
├── Cargo.toml
├── Cargo.lock
├── .cargo/
│   └── config.toml          # cargo configuration
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── config.rs
│   ├── error.rs
│   ├── handler/
│   │   ├── mod.rs
│   │   └── user.rs
│   ├── service/
│   │   ├── mod.rs
│   │   └── user.rs
│   └── model/
│       ├── mod.rs
│       └── user.rs
├── tests/
│   └── api_test.rs
├── Dockerfile
├── Makefile
└── README.md
```

### Dockerfile

```dockerfile
FROM rust:1.78-slim AS build
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release  # cache dependencies
COPY src ./src
RUN cargo build --release

FROM gcr.io/distroless/cc-debian12
COPY --from=build /app/target/release/myservice /
USER nonroot
ENTRYPOINT ["/myservice"]
```

The dependency-caching trick (`cargo build` with a dummy `main.rs` before copying actual source) avoids rebuilding all dependencies on every code change.

---

## Key Takeaways

- Builder pattern for complex construction, newtype for type safety, typestate for compile-time state machines.
- `serde` is the universal serialisation framework. `#[derive(Serialize, Deserialize)]` is all you need for most formats.
- `clap` with derive macros makes CLI argument parsing declarative and type-safe.
- `tokio` + `axum` + `sqlx` is the modern async web stack. `reqwest` for HTTP client.
- `anyhow` for applications, `thiserror` for libraries — the standard error handling pair.
- `tracing` has replaced `log` as the standard for structured, span-aware logging.
- Cache dependencies in Docker builds by building with a dummy source file before copying real code.
