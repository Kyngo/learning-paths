---
title: "Types & Variables"
weight: 2
---

# Types & Variables

Go is statically typed with type inference. Every variable has a fixed type determined at compile time, but you rarely need to write types explicitly thanks to the `:=` short declaration.

---

## Declaring Variables

### Long Form

```go
var name string = "Alice"
var age int = 30
var active bool = true
```

### Short Declaration (Most Common)

```go
name := "Alice"    // type inferred as string
age := 30          // type inferred as int
active := true     // type inferred as bool
```

`:=` declares and assigns in one step. It only works inside functions (not at package level).

### Multiple Declarations

```go
var (
    host    string = "localhost"
    port    int    = 8080
    debug   bool   = false
)

// Short form
x, y, z := 1, 2, 3
name, err := getUser(id)  // common pattern: value + error
```

---

## Basic Types

| Type | Size | Zero Value | Example |
|------|------|-----------|---------|
| `bool` | 1 byte | `false` | `true`, `false` |
| `int` | platform (32/64 bit) | `0` | `42`, `-7` |
| `int8` | 1 byte | `0` | -128 to 127 |
| `int16` | 2 bytes | `0` | -32768 to 32767 |
| `int32` | 4 bytes | `0` | alias: `rune` |
| `int64` | 8 bytes | `0` | |
| `uint` | platform | `0` | unsigned |
| `uint8` | 1 byte | `0` | alias: `byte` |
| `uint16` | 2 bytes | `0` | |
| `uint32` | 4 bytes | `0` | |
| `uint64` | 8 bytes | `0` | |
| `float32` | 4 bytes | `0.0` | ~7 decimal digits |
| `float64` | 8 bytes | `0.0` | ~15 decimal digits |
| `complex64` | 8 bytes | `(0+0i)` | float32 real + imag |
| `complex128` | 16 bytes | `(0+0i)` | float64 real + imag |
| `string` | varies | `""` | UTF-8 encoded |
| `byte` | 1 byte | `0` | alias for `uint8` |
| `rune` | 4 bytes | `0` | alias for `int32` (Unicode code point) |

---

## Zero Values

Every type in Go has a **zero value** — the value assigned if you declare a variable without initialising it. This is not null/undefined — it is a real, usable value.

```go
var i int       // 0
var f float64   // 0.0
var b bool      // false
var s string    // "" (empty string)
var p *int      // nil
var sl []int    // nil (but len(sl) == 0, can append to it)
var m map[string]int  // nil (must initialise before writing)
```

### Make the Zero Value Useful

This is a core Go principle. Types should be designed so that their zero value is valid:

```go
// sync.Mutex — zero value is an unlocked mutex
var mu sync.Mutex  // ready to use, no constructor needed

// bytes.Buffer — zero value is an empty buffer
var buf bytes.Buffer
buf.WriteString("hello")

// sync.WaitGroup — zero value is a group with count 0
var wg sync.WaitGroup
```

---

## Constants

Constants are immutable values known at compile time:

```go
const Pi = 3.14159265358979
const MaxRetries = 3
const AppName = "myservice"

const (
    StatusActive   = "active"
    StatusInactive = "inactive"
    StatusDeleted  = "deleted"
)
```

### Untyped Constants

Go constants can be "untyped" — they have a default type but adapt to context:

```go
const x = 42  // untyped integer constant

var i int = x       // works
var f float64 = x   // works (42.0)
var b byte = x      // works (42 fits in a byte)
```

### iota — The Constant Generator

`iota` is an auto-incrementing integer for enumeration:

```go
type Weekday int

const (
    Sunday Weekday = iota  // 0
    Monday                 // 1
    Tuesday                // 2
    Wednesday              // 3
    Thursday               // 4
    Friday                 // 5
    Saturday               // 6
)
```

**Common iota patterns:**

```go
// Bit flags
const (
    FlagRead    = 1 << iota  // 1
    FlagWrite                // 2
    FlagExecute              // 4
)

// Skip values
const (
    _  = iota  // skip 0
    KB = 1 << (10 * iota)  // 1024
    MB                      // 1048576
    GB                      // 1073741824
    TB                      // 1099511627776
)
```

---

## Type Conversions

Go has **no implicit type conversions**. Every conversion must be explicit:

```go
var i int = 42
var f float64 = float64(i)  // explicit conversion
var u uint = uint(f)        // explicit conversion

// This DOES NOT compile:
// var f float64 = i  // cannot use i (type int) as type float64
```

### String Conversions

```go
import "strconv"

// Integer ↔ string
s := strconv.Itoa(42)          // "42"
n, err := strconv.Atoi("42")  // 42, nil

// Float ↔ string
s := strconv.FormatFloat(3.14, 'f', 2, 64)  // "3.14"
f, err := strconv.ParseFloat("3.14", 64)     // 3.14, nil

// Bool ↔ string
s := strconv.FormatBool(true)  // "true"
b, err := strconv.ParseBool("true")  // true, nil

// fmt.Sprintf for formatting
s := fmt.Sprintf("User %d: %s", id, name)
```

---

## Strings, Bytes, and Runes

### Strings Are Immutable Byte Slices

```go
s := "Hello, 世界"

len(s)    // 13 (bytes, not characters — "世" and "界" are 3 bytes each)
s[0]      // 72 (byte value of 'H')
s[7:]     // "世界"
```

**Strings are UTF-8 encoded by default.** ASCII characters are 1 byte; many Unicode characters are 2-4 bytes.

### Runes vs Bytes

| Type | Represents | Example |
|------|-----------|---------|
| `byte` (uint8) | A single byte | `'A'` (65) |
| `rune` (int32) | A Unicode code point | `'世'` (19990) |

```go
s := "Hello, 世界"

// Iterating by byte — WRONG for Unicode
for i := 0; i < len(s); i++ {
    fmt.Printf("%c ", s[i])  // garbled output for multi-byte chars
}

// Iterating by rune — CORRECT
for i, r := range s {
    fmt.Printf("%d:%c ", i, r)
    // 0:H 1:e 2:l 3:l 4:o 5:, 6:  7:世 10:界
}

// Convert to rune slice for character-level operations
runes := []rune(s)
len(runes)    // 9 (characters)
runes[7]      // '世'
```

### String Builder (Efficient Concatenation)

```go
// BAD — O(n²) because strings are immutable
s := ""
for _, word := range words {
    s += word + " "  // allocates a new string each time
}

// GOOD — O(n) with strings.Builder
var b strings.Builder
for _, word := range words {
    b.WriteString(word)
    b.WriteString(" ")
}
result := b.String()
```

---

## Arrays and Slices

### Arrays (Fixed Length — Rarely Used Directly)

```go
var a [5]int                    // [0, 0, 0, 0, 0]
b := [3]string{"a", "b", "c"}  // ["a", "b", "c"]
c := [...]int{1, 2, 3}         // length inferred: [1, 2, 3]
```

Arrays have a fixed size that is part of the type: `[3]int` and `[5]int` are different types.

### Slices (Dynamic — The Standard Choice)

```go
s := []int{1, 2, 3, 4, 5}
s = append(s, 6)        // grow
sub := s[1:4]           // [2, 3, 4] — a view into s
len(s)                  // 6
cap(s)                  // capacity (internal array size)

// Make with length and capacity
s := make([]int, 0, 100)  // length 0, preallocated for 100
```

### Slice Gotcha: Shared Backing Array

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]  // [2, 3] — shares memory with a

b[0] = 99
fmt.Println(a)  // [1, 99, 3, 4, 5] — a was mutated!

// To avoid: copy the slice
c := make([]int, len(b))
copy(c, b)
```

---

## Maps

```go
// Create
m := map[string]int{
    "alice": 30,
    "bob":   25,
}

// Access
age := m["alice"]    // 30
age := m["unknown"]  // 0 (zero value, not an error)

// Check existence
age, ok := m["alice"]
if !ok {
    fmt.Println("not found")
}

// Add/update
m["carol"] = 28

// Delete
delete(m, "bob")

// Iterate (order is random)
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}

// Length
len(m)  // number of entries
```

**Warning:** A nil map can be read (returns zero values) but writing to it panics. Always initialise with `make(map[K]V)` or a literal.

---

## Pointers

Go has pointers but no pointer arithmetic:

```go
x := 42
p := &x     // p is *int, points to x
*p = 100    // dereference: x is now 100
fmt.Println(x)  // 100
```

### When to Use Pointers

| Use Pointer | Use Value |
|-------------|-----------|
| Large structs (avoid copying) | Small structs (≤ few fields) |
| Need to mutate the original | Read-only access |
| Optional values (`nil` = absent) | Always present |
| Implementing interfaces with mutation | Immutable value types |

```go
// Pointer receiver — can modify the struct
func (u *User) SetName(name string) {
    u.Name = name
}

// Value receiver — works on a copy
func (u User) FullName() string {
    return u.FirstName + " " + u.LastName
}
```

### `new` vs `make`

| Function | Returns | Use For |
|----------|---------|---------|
| `new(T)` | `*T` (pointer to zero value) | Any type |
| `make(T, ...)` | `T` (initialised value) | Slices, maps, channels only |

```go
p := new(int)        // *int, points to 0
s := make([]int, 10) // []int with length 10
m := make(map[string]int) // initialised empty map
ch := make(chan int, 5)   // buffered channel
```

---

## Key Takeaways

- Go is statically typed with type inference. Use `:=` for short declarations inside functions.
- Every type has a zero value. Design your types so the zero value is useful.
- No implicit type conversions — all conversions must be explicit.
- Strings are UTF-8 byte slices. Use `range` to iterate by rune, not by byte.
- Slices share backing arrays — mutating a sub-slice mutates the original. Copy when independence is needed.
- Maps must be initialised before writing. Always check existence with the comma-ok idiom.
- Pointers enable mutation and avoid copies, but Go has no pointer arithmetic.
