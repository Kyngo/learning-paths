---
title: "Generics"
weight: 9
---

# Generics

Go 1.18 (2022) added generics — type parameters that let you write functions and types that work with multiple types while preserving type safety. Go's generics are deliberately simpler than those in Java, Rust, or Haskell.

---

## The Problem Generics Solve

Before generics, you had three options for type-agnostic code:

```go
// 1. Copy-paste for each type
func MinInt(a, b int) int { ... }
func MinFloat64(a, b float64) float64 { ... }
func MinString(a, b string) string { ... }

// 2. Use interface{}/any — loses type safety
func Min(a, b any) any { ... }  // caller must type-assert

// 3. Code generation — complex tooling
//go:generate ...
```

With generics:

```go
func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

Min(3, 5)           // int
Min(3.14, 2.71)     // float64
Min("alice", "bob") // string
```

---

## Type Parameters

### Functions

```go
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

Contains([]int{1, 2, 3}, 2)           // true
Contains([]string{"a", "b"}, "c")     // false
```

### Types

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// Usage
s := &Stack[int]{}
s.Push(1)
s.Push(2)
v, _ := s.Pop()  // 2
```

---

## Constraints

Constraints restrict what types can be used with a type parameter. They are expressed as interfaces.

### Built-in Constraints

| Constraint | Allows | Package |
|-----------|--------|---------|
| `any` | Any type | builtin |
| `comparable` | Types supporting `==` and `!=` | builtin |
| `cmp.Ordered` | Types supporting `<`, `>`, `<=`, `>=` | `cmp` |
| `constraints.Integer` | All integer types | `golang.org/x/exp/constraints` |
| `constraints.Float` | All float types | `golang.org/x/exp/constraints` |
| `constraints.Signed` | Signed integers | `golang.org/x/exp/constraints` |

### Custom Constraints

```go
// A constraint is just an interface with type elements
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~float32 | ~float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

### The `~` Operator

`~T` matches T and any type whose underlying type is T:

```go
type Celsius float64
type Fahrenheit float64

type Temperature interface {
    ~float64  // matches float64, Celsius, Fahrenheit
}

func Convert[T Temperature](t T) T {
    return t * 1.8 + 32  // works with any float64-based type
}
```

Without `~`, only the exact type `float64` would match, not `Celsius` or `Fahrenheit`.

---

## Useful Generic Functions

### Map

```go
func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

names := Map(users, func(u User) string { return u.Name })
```

### Filter

```go
func Filter[T any](slice []T, predicate func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

adults := Filter(users, func(u User) bool { return u.Age >= 18 })
```

### Reduce

```go
func Reduce[T, U any](slice []T, initial U, f func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = f(result, v)
    }
    return result
}

total := Reduce(prices, 0.0, func(sum, p float64) float64 { return sum + p })
```

### Keys / Values

```go
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}
```

---

## Standard Library Generics

Go 1.21+ includes several generic packages:

### `slices` Package

```go
import "slices"

slices.Contains([]int{1, 2, 3}, 2)  // true
slices.Sort([]int{3, 1, 2})         // [1, 2, 3]
slices.Reverse([]int{1, 2, 3})      // [3, 2, 1]
slices.Index([]int{1, 2, 3}, 2)     // 1
slices.Compact([]int{1, 1, 2, 3, 3}) // [1, 2, 3]
slices.Max([]int{1, 5, 3})          // 5
slices.Min([]int{1, 5, 3})          // 1
```

### `maps` Package

```go
import "maps"

maps.Keys(m)       // all keys
maps.Values(m)     // all values
maps.Clone(m)      // shallow copy
maps.Equal(m1, m2) // element-wise comparison
maps.DeleteFunc(m, func(k string, v int) bool { return v == 0 })
```

### `cmp` Package

```go
import "cmp"

cmp.Compare(1, 2)      // -1
cmp.Compare("b", "a")  // 1
cmp.Or(0, 0, 42, 7)    // 42 (first non-zero value)
```

---

## When NOT to Use Generics

Go's philosophy is simplicity. Generics add complexity. The Go team recommends:

| Use Generics | Don't Use Generics |
|-------------|-------------------|
| Containers (Stack, Queue, Set) | Methods where the type is always known |
| Utility functions (Map, Filter, Contains) | When `any` (interface{}) suffices |
| Type-safe wrappers | When the abstraction doesn't save code |
| When you are writing the same logic 3+ times for different types | When it makes the code harder to read |

### Guidelines

1. **Write concrete code first.** Only generalise when you have 3+ near-identical implementations.
2. **Don't abstract prematurely.** A function that works on `[]User` is clearer than `[]T` if you only have users.
3. **Prefer interfaces over generics** when the constraint is behavioural (method-based).
4. **Generics are for types, interfaces are for behaviour.**

---

## Key Takeaways

- Generics let you write type-safe, reusable functions and data structures without code duplication or `any`.
- Constraints are interfaces with type elements. Use `any` for no constraint, `comparable` for equality, `cmp.Ordered` for comparison.
- The `~` operator matches underlying types — essential for user-defined types based on primitives.
- The `slices` and `maps` packages provide ready-made generic utilities. Use them before writing your own.
- Don't reach for generics by default. Write concrete code first, generalise only when the duplication is clear and the abstraction simplifies the code.
