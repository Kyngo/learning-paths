---
title: "Control Flow"
weight: 3
---

# Control Flow

Go has minimal control flow constructs: `if`, `for`, `switch`, and `defer`. There is no `while`, no `do-while`, no ternary operator, and no `try/catch`. This simplicity is intentional.

---

## If/Else

```go
if x > 0 {
    fmt.Println("positive")
} else if x == 0 {
    fmt.Println("zero")
} else {
    fmt.Println("negative")
}
```

### If with Init Statement

Go allows a short statement before the condition — scoped to the if/else block:

```go
if err := doSomething(); err != nil {
    return fmt.Errorf("failed: %w", err)
}
// err is not accessible here
```

This is the idiomatic error-checking pattern. You will write it hundreds of times.

### No Ternary Operator

Go intentionally omits the ternary `? :` operator. Use an if/else:

```go
// Other languages: result = condition ? a : b
// Go:
var result string
if condition {
    result = a
} else {
    result = b
}
```

---

## For (The Only Loop)

Go has exactly one loop keyword: `for`. It covers all loop patterns.

### Traditional For

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

### While-Style

```go
for condition {
    // runs while condition is true
}
```

### Infinite Loop

```go
for {
    // runs forever (use break or return to exit)
}
```

### Range (Iterating Collections)

```go
// Slice
numbers := []int{10, 20, 30}
for i, v := range numbers {
    fmt.Printf("index %d: value %d\n", i, v)
}

// Map
ages := map[string]int{"alice": 30, "bob": 25}
for key, value := range ages {
    fmt.Printf("%s is %d\n", key, value)
}

// String (iterates by rune, not byte)
for i, r := range "Hello, 世界" {
    fmt.Printf("%d: %c\n", i, r)
}

// Channel
for msg := range ch {
    process(msg)  // runs until channel is closed
}

// Range over integer (Go 1.22+)
for i := range 10 {
    fmt.Println(i)  // 0 through 9
}
```

### Ignoring Values

```go
for _, v := range slice {  // ignore index
    fmt.Println(v)
}

for i := range slice {  // ignore value (index only)
    fmt.Println(i)
}
```

### Break and Continue

```go
for i := 0; i < 100; i++ {
    if i%2 == 0 {
        continue  // skip even numbers
    }
    if i > 50 {
        break  // stop at 50
    }
    fmt.Println(i)
}
```

### Labels (Breaking Outer Loops)

```go
outer:
    for i := 0; i < 10; i++ {
        for j := 0; j < 10; j++ {
            if i+j > 15 {
                break outer  // breaks the outer loop
            }
        }
    }
```

---

## Switch

Go's switch is more powerful than most languages:

```go
switch day {
case "Monday":
    fmt.Println("start of week")
case "Friday":
    fmt.Println("almost weekend")
case "Saturday", "Sunday":
    fmt.Println("weekend")
default:
    fmt.Println("midweek")
}
```

### Key Differences from C/Java

| Feature | Go | C/Java |
|---------|-----|--------|
| Fallthrough | Must be explicit (`fallthrough`) | Implicit (need `break`) |
| Multiple values per case | `case "a", "b":` | Separate cases |
| Expression cases | Any expression | Constants only (Java < 14) |
| No-condition switch | `switch { case x > 0: ... }` | N/A |

### Switch with Init Statement

```go
switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("macOS")
case "linux":
    fmt.Println("Linux")
default:
    fmt.Printf("Other: %s\n", os)
}
```

### Expressionless Switch (If-Else Chain Replacement)

```go
switch {
case t.Hour() < 12:
    fmt.Println("morning")
case t.Hour() < 17:
    fmt.Println("afternoon")
default:
    fmt.Println("evening")
}
```

### Type Switch

```go
switch v := val.(type) {
case int:
    fmt.Printf("int: %d\n", v)
case string:
    fmt.Printf("string: %s\n", v)
case bool:
    fmt.Printf("bool: %t\n", v)
default:
    fmt.Printf("unknown: %T\n", v)
}
```

---

## Defer

`defer` schedules a function call to run when the enclosing function returns. Deferred calls are executed in **LIFO order** (last in, first out).

```go
func readFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()  // guaranteed to run when function returns

    return io.ReadAll(f)
}
```

### Common Uses

```go
// Resource cleanup
defer file.Close()
defer conn.Close()
defer rows.Close()
defer mu.Unlock()

// Timing
func operation() {
    start := time.Now()
    defer func() {
        fmt.Printf("took %v\n", time.Since(start))
    }()
    // ... work ...
}

// Recovering from panics
defer func() {
    if r := recover(); r != nil {
        log.Printf("recovered: %v", r)
    }
}()
```

### Defer Gotchas

**Arguments are evaluated immediately:**

```go
x := 10
defer fmt.Println(x)  // prints 10, not whatever x is later
x = 20
```

**Defers in a loop accumulate (don't close files in a loop with defer):**

```go
// BAD — all files open until function returns
for _, path := range paths {
    f, _ := os.Open(path)
    defer f.Close()  // not closed until outer function returns
}

// GOOD — extract to a function
for _, path := range paths {
    processFile(path)  // deferred close runs when processFile returns
}
```

---

## Goto

Go has `goto` but it is rarely used:

```go
    // Legal but discouraged
    goto cleanup
    // ...
cleanup:
    // cleanup code
```

The only legitimate use is breaking out of deeply nested logic in generated code. In hand-written code, prefer named returns, extract to a function, or use labels with `break`.

---

## Key Takeaways

- `for` is the only loop. It handles traditional, while-style, infinite, and range-based iteration.
- `if` supports an init statement (scoped variable) — use it for the `err != nil` pattern.
- `switch` does not fall through by default (opposite of C). Cases can be expressions, not just constants.
- `defer` guarantees cleanup on function return. Deferred calls run in LIFO order.
- Defer arguments are evaluated at the defer statement, not when the deferred function runs.
- No ternary operator, no while keyword, no do-while. Simplicity is the design goal.
