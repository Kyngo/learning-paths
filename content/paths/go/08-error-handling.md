---
title: "Error Handling"
weight: 8
---

# Error Handling

Go does not have exceptions. Errors are values — returned from functions, inspected by callers, and handled explicitly. This is a deliberate design choice that makes error paths visible and forces you to think about failure at every call site.

---

## The `error` Interface

```go
type error interface {
    Error() string
}
```

Any type with an `Error() string` method is an error. The most common way to create errors:

```go
import "errors"
import "fmt"

// Simple error
err := errors.New("something went wrong")

// Formatted error
err := fmt.Errorf("user %d not found", userID)
```

---

## The Error Return Pattern

Functions that can fail return an error as the last value:

```go
func ReadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading config: %w", err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parsing config: %w", err)
    }

    return &cfg, nil
}
```

### The `if err != nil` Pattern

```go
result, err := doSomething()
if err != nil {
    return fmt.Errorf("context: %w", err)
}
// use result
```

You will write this pattern thousands of times. It is verbose by design — error handling is not hidden in a catch block somewhere else.

---

## Wrapping Errors

Since Go 1.13, `%w` in `fmt.Errorf` wraps an error, preserving the original:

```go
func fetchUser(id int) (*User, error) {
    row := db.QueryRow("SELECT * FROM users WHERE id = $1", id)
    var u User
    if err := row.Scan(&u.ID, &u.Name, &u.Email); err != nil {
        return nil, fmt.Errorf("fetchUser(%d): %w", id, err)
    }
    return &u, nil
}
```

The call chain builds a stack of context:

```
fetchUser(42): sql: no rows in result set
```

### Unwrapping Errors

```go
// errors.Is — checks if any error in the chain matches a target
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrNotFound
}

// errors.As — extracts a specific error type from the chain
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("failed path:", pathErr.Path)
}
```

| Function | Use |
|----------|-----|
| `errors.Is(err, target)` | Check for a specific error value (sentinel) |
| `errors.As(err, &target)` | Extract a specific error type |
| `errors.Unwrap(err)` | Get the wrapped error (one level) |

---

## Sentinel Errors

Predefined error values for well-known conditions:

```go
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorised = errors.New("unauthorised")
    ErrConflict     = errors.New("conflict")
)

func GetUser(id int) (*User, error) {
    u, err := repo.FindByID(id)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("GetUser: %w", err)
    }
    return u, nil
}

// Caller checks
if errors.Is(err, ErrNotFound) {
    http.Error(w, "user not found", 404)
}
```

### Standard Library Sentinels

| Package | Sentinel | Meaning |
|---------|----------|---------|
| io | `io.EOF` | End of input |
| sql | `sql.ErrNoRows` | Query returned no results |
| context | `context.Canceled` | Context was cancelled |
| context | `context.DeadlineExceeded` | Timeout |
| os | `os.ErrNotExist` | File does not exist |
| os | `os.ErrPermission` | Permission denied |

---

## Custom Error Types

When you need to carry additional information:

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s — %s", e.Field, e.Message)
}

func ValidateUser(u *User) error {
    if u.Email == "" {
        return &ValidationError{Field: "email", Message: "required"}
    }
    if len(u.Name) < 2 {
        return &ValidationError{Field: "name", Message: "too short"}
    }
    return nil
}

// Caller extracts the type
var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Printf("field %s: %s\n", ve.Field, ve.Message)
}
```

### Multi-Error

Collect multiple errors (e.g., validating all fields at once):

```go
// Go 1.20+ — errors.Join
func ValidateUser(u *User) error {
    var errs []error
    if u.Email == "" {
        errs = append(errs, &ValidationError{Field: "email", Message: "required"})
    }
    if u.Name == "" {
        errs = append(errs, &ValidationError{Field: "name", Message: "required"})
    }
    return errors.Join(errs...)  // nil if no errors
}
```

---

## Panic and Recover

### Panic

`panic` immediately stops the current function, runs deferred functions, then unwinds the stack:

```go
panic("something terrible happened")
panic(fmt.Errorf("invalid state: %v", state))
```

### When to Panic

| Panic | Error Return |
|-------|-------------|
| Programmer error (bug, impossible state) | Expected failure (file not found, network error) |
| Uninitialised required dependency | User input validation failure |
| Violated invariant that should never happen | Business logic failure |

```go
// Panic is appropriate here — this is a bug
func MustParse(s string) *Template {
    t, err := Parse(s)
    if err != nil {
        panic(fmt.Sprintf("MustParse: %v", err))
    }
    return t
}

// "Must" prefix convention = panics on error (used in stdlib: template.Must, regexp.MustCompile)
```

### Recover

`recover` catches a panic, but only inside a deferred function:

```go
func safeOperation() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()

    riskyOperation()  // might panic
    return nil
}
```

**Use recover sparingly.** It is appropriate in:
- HTTP handlers (prevent one request from crashing the server)
- Plugin/extension systems (isolate untrusted code)
- Top-level goroutines (log and continue)

---

## Error Handling Patterns

### Wrap with Context

```go
if err != nil {
    return fmt.Errorf("loading user %d: %w", id, err)
}
```

### Handle Once, at the Right Level

```go
// LOW LEVEL — return error with context
func readFile(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("readFile %s: %w", path, err)
    }
    return data, nil
}

// HIGH LEVEL — decide what to do (log, retry, return HTTP error)
func handleRequest(w http.ResponseWriter, r *http.Request) {
    cfg, err := readFile("config.json")
    if err != nil {
        log.Printf("ERROR: %v", err)
        http.Error(w, "internal error", 500)
        return
    }
    // use cfg
}
```

### Don't Log and Return

```go
// BAD — logs the error AND returns it (caller might log again)
if err != nil {
    log.Printf("error: %v", err)
    return err
}

// GOOD — either log OR return, not both
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}
```

---

## Key Takeaways

- Errors are values, not exceptions. Return them, wrap them with context, check them.
- `%w` wraps errors for later inspection with `errors.Is` and `errors.As`.
- Sentinel errors (`var ErrNotFound = errors.New(...)`) are for well-known conditions that callers check by identity.
- Custom error types carry structured data (`ValidationError`, `HTTPError`).
- `panic` is for programmer errors and impossible states. `recover` catches panics in deferred functions.
- Don't log and return the same error. Handle it once, at the appropriate level.
- The `if err != nil { return }` pattern is verbose on purpose — it makes error paths explicit and visible.
