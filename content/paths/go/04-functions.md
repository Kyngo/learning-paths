---
title: "Functions"
weight: 4
---

# Functions

Functions in Go are first-class values. They can be assigned to variables, passed as arguments, returned from other functions, and defined inline as closures. Go functions support multiple return values — the foundation of Go's error handling model.

---

## Basic Functions

```go
func add(a int, b int) int {
    return a + b
}

// Shortened: consecutive params of same type
func add(a, b int) int {
    return a + b
}
```

---

## Multiple Return Values

Go functions can return multiple values. The dominant pattern is `(result, error)`:

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// Caller must handle both
result, err := divide(10, 3)
if err != nil {
    log.Fatal(err)
}
fmt.Println(result)
```

### Named Return Values

```go
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = fmt.Errorf("division by zero")
        return  // naked return — returns named values
    }
    result = a / b
    return
}
```

Named returns serve as documentation. **Naked returns** (returning without arguments) are acceptable only in short functions — in longer functions they reduce readability.

---

## Variadic Functions

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

sum(1, 2, 3)      // 6
sum(1, 2, 3, 4, 5) // 15

// Expand a slice into variadic args
numbers := []int{1, 2, 3}
sum(numbers...)
```

The variadic parameter must be the last parameter. Inside the function, it is a slice.

---

## Functions as Values

Functions are first-class citizens:

```go
// Assign to a variable
greet := func(name string) string {
    return "Hello, " + name
}
fmt.Println(greet("Alice"))

// Pass as argument
func apply(f func(int) int, value int) int {
    return f(value)
}

double := func(x int) int { return x * 2 }
result := apply(double, 5)  // 10
```

### Function Types

```go
// Define a function type for readability
type Transformer func(int) int

func apply(f Transformer, value int) int {
    return f(value)
}
```

---

## Closures

A closure captures variables from its enclosing scope:

```go
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

c := counter()
fmt.Println(c())  // 1
fmt.Println(c())  // 2
fmt.Println(c())  // 3
```

### Closure Gotcha: Loop Variables

```go
// BUG (before Go 1.22): all goroutines see the same i
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i)  // prints 5, 5, 5, 5, 5
    }()
}

// FIX (pre-1.22): capture as parameter
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)  // prints 0, 1, 2, 3, 4 (in some order)
    }(i)
}

// Go 1.22+: loop variable is per-iteration (fixed by default)
```

---

## Common Patterns

### Option Functions

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
}

type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}

func NewServer(host string, opts ...Option) *Server {
    s := &Server{
        host:    host,
        port:    8080,           // default
        timeout: 30 * time.Second, // default
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
s := NewServer("localhost",
    WithPort(9090),
    WithTimeout(10*time.Second),
)
```

### Middleware / Decorator

```go
type Handler func(http.ResponseWriter, *http.Request)

func withLogging(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next(w, r)
    }
}

func withAuth(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        if !isAuthenticated(r) {
            http.Error(w, "unauthorized", 401)
            return
        }
        next(w, r)
    }
}
```

### Predicate Functions

```go
func filter[T any](items []T, predicate func(T) bool) []T {
    var result []T
    for _, item := range items {
        if predicate(item) {
            result = append(result, item)
        }
    }
    return result
}

evens := filter([]int{1, 2, 3, 4, 5}, func(n int) bool {
    return n%2 == 0
})
```

---

## The `init()` Function

Every package can define `init()` functions that run automatically during package initialisation:

```go
package database

var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
}
```

### Execution Order

1. Imported packages are initialised first (recursively)
2. Package-level variables are initialised
3. `init()` functions run (in the order they appear in source files)
4. `main()` runs

**Use `init()` sparingly.** It makes dependencies implicit and testing harder. Prefer explicit initialisation in `main()`.

---

## Key Takeaways

- Multiple return values enable the `(value, error)` pattern — Go's primary error handling mechanism.
- Functions are first-class values: assign to variables, pass as arguments, return from functions.
- Closures capture surrounding variables. Watch for the loop variable gotcha (fixed in Go 1.22).
- Named return values document what a function returns. Naked returns are only clear in short functions.
- Functional options (`WithXxx`) are the idiomatic way to build constructors with optional parameters.
- `init()` runs before `main()` but makes dependencies implicit — prefer explicit initialisation.
