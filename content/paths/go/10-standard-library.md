---
title: "Standard Library"
weight: 10
---

# Standard Library

Go's standard library is one of its greatest strengths — comprehensive enough that many production services run with zero third-party dependencies. This section covers the packages you will use most often.

---

## `fmt` — Formatted I/O

```go
fmt.Println("Hello")                        // print with newline
fmt.Printf("Name: %s, Age: %d\n", name, age) // formatted output
s := fmt.Sprintf("User %d", id)              // format to string
fmt.Fprintf(w, "Response: %s", body)          // format to writer
```

### Format Verbs

| Verb | Description | Example |
|------|-------------|---------|
| `%v` | Default format | `{Alice 30}` |
| `%+v` | With field names | `{Name:Alice Age:30}` |
| `%#v` | Go-syntax representation | `main.User{Name:"Alice", Age:30}` |
| `%T` | Type | `main.User` |
| `%d` | Integer (decimal) | `42` |
| `%b` | Integer (binary) | `101010` |
| `%x` | Integer (hex) | `2a` |
| `%f` | Float | `3.140000` |
| `%.2f` | Float (2 decimals) | `3.14` |
| `%s` | String | `hello` |
| `%q` | Quoted string | `"hello"` |
| `%p` | Pointer | `0xc000012080` |
| `%w` | Wrap error (Errorf only) | — |

---

## `net/http` — HTTP Server and Client

### HTTP Server

```go
mux := http.NewServeMux()

mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
})

mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // Go 1.22+ path parameters
    fmt.Fprintf(w, "User: %s", id)
})

mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }
    // process user
    w.WriteHeader(http.StatusCreated)
})

server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}
log.Fatal(server.ListenAndServe())
```

### HTTP Client

```go
client := &http.Client{
    Timeout: 10 * time.Second,
}

resp, err := client.Get("https://api.example.com/users")
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("unexpected status: %d", resp.StatusCode)
}

var users []User
if err := json.NewDecoder(resp.Body).Decode(&users); err != nil {
    return err
}
```

**Always set a timeout on `http.Client`.** The default client has no timeout and will hang forever on a slow server.

---

## `encoding/json` — JSON

### Marshal (Go → JSON)

```go
user := User{ID: 1, Name: "Alice", Email: "alice@example.com"}

data, err := json.Marshal(user)
// {"id":1,"name":"Alice","email":"alice@example.com"}

// Pretty print
data, err := json.MarshalIndent(user, "", "  ")
```

### Unmarshal (JSON → Go)

```go
var user User
err := json.Unmarshal(data, &user)

// From a reader (HTTP body, file):
err := json.NewDecoder(r.Body).Decode(&user)
```

### JSON Struct Tags

```go
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`  // omit if empty
    Pass  string `json:"-"`                // never marshal
}
```

### Dynamic JSON

```go
// Unknown structure
var data map[string]any
json.Unmarshal(raw, &data)

// Access
name := data["name"].(string)
age := data["age"].(float64)  // JSON numbers are float64
```

---

## `context` — Cancellation and Deadlines

Context carries deadlines, cancellation signals, and request-scoped values across API boundaries.

```go
// With timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()  // always defer cancel

// With cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// With value (use sparingly)
ctx = context.WithValue(ctx, "requestID", "abc-123")

// Checking cancellation
select {
case <-ctx.Done():
    return ctx.Err()  // context.Canceled or context.DeadlineExceeded
case result := <-doWork(ctx):
    return result
}
```

### Context Rules

1. Pass `context.Context` as the **first parameter** of every function that accepts it
2. **Never** store context in a struct
3. Use `context.Background()` at the top level (main, tests)
4. Use `context.TODO()` when you know a context is needed but haven't plumbed it yet
5. Always `defer cancel()` when creating a derived context

---

## `testing` — Test Framework

```go
// user_test.go
package user

import "testing"

func TestFullName(t *testing.T) {
    u := User{FirstName: "Alice", LastName: "Chen"}
    got := u.FullName()
    want := "Alice Chen"
    if got != want {
        t.Errorf("FullName() = %q, want %q", got, want)
    }
}

// Table-driven test
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        valid bool
    }{
        {"valid", "user@example.com", true},
        {"no domain", "user@", false},
        {"empty", "", false},
        {"no at", "userexample.com", false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            if got != tt.valid {
                t.Errorf("ValidateEmail(%q) = %v, want %v", tt.email, got, tt.valid)
            }
        })
    }
}
```

```bash
go test ./...           # run all tests
go test -v ./...        # verbose
go test -run TestFull   # run matching tests
go test -race ./...     # with race detector
go test -count=1 ./...  # disable test caching
go test -cover ./...    # show coverage
```

---

## `log/slog` — Structured Logging (Go 1.21+)

```go
import "log/slog"

slog.Info("user created",
    slog.String("name", user.Name),
    slog.Int("id", user.ID),
)
// time=2024-01-15T10:30:00Z level=INFO msg="user created" name=Alice id=42

slog.Error("failed to connect",
    slog.String("host", host),
    slog.Any("error", err),
)

// JSON output
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)
```

---

## `time` — Time and Duration

```go
now := time.Now()
later := now.Add(24 * time.Hour)
duration := later.Sub(now)

// Formatting — uses the reference time: Mon Jan 2 15:04:05 MST 2006
s := now.Format("2006-01-02T15:04:05Z07:00")  // ISO 8601
t, err := time.Parse("2006-01-02", "2024-01-15")

// Predefined layouts
time.RFC3339     // "2006-01-02T15:04:05Z07:00"
time.DateTime    // "2006-01-02 15:04:05"
time.DateOnly    // "2006-01-02"
time.TimeOnly    // "15:04:05"
```

**Go's reference time is January 2, 2006 at 15:04:05 MST (Mountain Standard Time).** The values 1, 2, 3, 4, 5, 6, 7 help you remember: `01/02 03:04:05 PM '06 -0700`.

---

## `io` — I/O Primitives

```go
// Copy from reader to writer
io.Copy(dst, src)

// Read all
data, err := io.ReadAll(reader)

// Limit reading
limited := io.LimitReader(reader, 1024*1024)  // max 1 MB

// Pipe
pr, pw := io.Pipe()
go func() {
    pw.Write([]byte("hello"))
    pw.Close()
}()
data, _ := io.ReadAll(pr)
```

---

## `os` — Operating System Interface

```go
// Environment
os.Getenv("DATABASE_URL")
os.Setenv("DEBUG", "true")

// Files
data, err := os.ReadFile("config.json")
err := os.WriteFile("output.txt", data, 0644)

f, err := os.Create("output.txt")
defer f.Close()
f.Write([]byte("hello"))

// Process
os.Exit(1)
os.Args  // command-line arguments
```

---

## Key Takeaways

- The standard library covers HTTP (server + client), JSON, testing, logging, time, I/O, and more — often no third-party packages needed.
- Always set timeouts on `http.Client` and `http.Server`. The defaults are dangerous.
- Use `json.NewDecoder` for streams (HTTP bodies) and `json.Unmarshal` for byte slices.
- Context carries cancellation, deadlines, and request-scoped values. Always pass as the first parameter.
- `log/slog` (Go 1.21+) provides structured logging with levels — prefer it over `log.Printf`.
- Go's time formatting uses the reference date `2006-01-02 15:04:05` — memorise it or use the predefined constants.
