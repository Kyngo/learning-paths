---
title: "Interfaces"
weight: 6
---

# Interfaces

Interfaces are Go's primary abstraction mechanism. Unlike Java or C#, Go interfaces are satisfied **implicitly** — a type implements an interface simply by having the right methods. No `implements` keyword. No declaration of intent.

---

## Defining Interfaces

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}
```

An interface is a set of method signatures. Any type that has those methods satisfies the interface.

---

## Implicit Satisfaction

```go
type File struct { /* ... */ }

func (f *File) Read(p []byte) (n int, err error) { /* ... */ }
func (f *File) Write(p []byte) (n int, err error) { /* ... */ }

// File implicitly satisfies Reader, Writer, and ReadWriter
// No "implements" declaration needed
```

This decouples the interface definition from the implementation. You can define an interface in one package and satisfy it from another without any import dependency.

### Compile-Time Check

Force a compile-time check that a type satisfies an interface:

```go
var _ Reader = (*File)(nil)      // compile error if File doesn't implement Reader
var _ io.ReadWriter = (*File)(nil)
```

---

## The Empty Interface

```go
interface{}  // or 'any' (alias since Go 1.18)
```

The empty interface has no methods, so every type satisfies it. Use it when you need to accept any type:

```go
func Print(v any) {
    fmt.Println(v)
}

Print(42)
Print("hello")
Print([]int{1, 2, 3})
```

**Avoid overusing `any`.** It discards type safety. Prefer specific interfaces or generics.

---

## Type Assertions

Extract the concrete type from an interface value:

```go
var r Reader = &File{name: "data.txt"}

// Type assertion — panics if wrong
f := r.(*File)

// Safe type assertion — returns ok=false if wrong
f, ok := r.(*File)
if !ok {
    fmt.Println("not a *File")
}
```

### Type Switch

```go
func describe(v any) string {
    switch t := v.(type) {
    case int:
        return fmt.Sprintf("integer: %d", t)
    case string:
        return fmt.Sprintf("string: %q", t)
    case bool:
        return fmt.Sprintf("boolean: %t", t)
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("other: %T", t)
    }
}
```

---

## Key Standard Library Interfaces

### `error`

```go
type error interface {
    Error() string
}
```

The most used interface in Go. Any type with an `Error() string` method is an error.

### `fmt.Stringer`

```go
type Stringer interface {
    String() string
}
```

Implement this for custom string representation (like `toString()` in Java):

```go
func (u User) String() string {
    return fmt.Sprintf("%s %s <%s>", u.FirstName, u.LastName, u.Email)
}
```

### `io.Reader` and `io.Writer`

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

The composition primitives of Go I/O. Files, network connections, HTTP bodies, buffers, compressors, encrypters — all implement these interfaces. This is why you can pipe anything to anything.

```go
// Copy from any Reader to any Writer
io.Copy(os.Stdout, resp.Body)
io.Copy(file, gzipReader)
io.Copy(hashWriter, file)
```

### `io.Closer`

```go
type Closer interface {
    Close() error
}
```

### `sort.Interface`

```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

Implement these three methods to make any collection sortable:

```go
type ByAge []User

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

sort.Sort(ByAge(users))

// Or use sort.Slice (simpler, since Go 1.8):
sort.Slice(users, func(i, j int) bool {
    return users[i].Age < users[j].Age
})
```

### `http.Handler`

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

Any type with a `ServeHTTP` method can handle HTTP requests. The `http.HandlerFunc` adapter lets you use plain functions too:

```go
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(200)
    w.Write([]byte("ok"))
})
```

---

## Interface Design Principles

### Keep Interfaces Small

> "The bigger the interface, the weaker the abstraction." — Rob Pike

```go
// GOOD — small, focused
type Reader interface {
    Read(p []byte) (n int, err error)
}

// BAD — kitchen sink
type DataStore interface {
    Read(id string) ([]byte, error)
    Write(id string, data []byte) error
    Delete(id string) error
    List(prefix string) ([]string, error)
    Watch(id string) (<-chan Event, error)
    Backup(path string) error
    Restore(path string) error
}
```

### Define Interfaces Where They Are Used (Consumer Side)

```go
// In the consumer package, not the provider package
package handler

type UserRepository interface {
    FindByID(ctx context.Context, id int) (*User, error)
}

type Handler struct {
    repo UserRepository  // depends on the interface, not a concrete type
}
```

This is the opposite of Java's approach (interfaces defined by the provider). In Go, the consumer defines what it needs.

### Accept Interfaces, Return Structs

```go
// GOOD — function accepts an interface
func ProcessData(r io.Reader) error { ... }

// GOOD — function returns a concrete type
func NewServer(addr string) *Server { ... }

// AVOID — returning an interface hides the concrete type
func NewServer(addr string) ServerInterface { ... }
```

---

## Interface Values (Internals)

An interface value is a tuple of `(type, value)`:

```go
var r io.Reader         // (nil, nil) — nil interface
r = os.Stdin            // (*os.File, pointer to stdin)
r = &bytes.Buffer{}     // (*bytes.Buffer, pointer to buffer)
r = strings.NewReader("hello")  // (*strings.Reader, pointer to reader)
```

### The Nil Interface Trap

```go
var p *File = nil
var r io.Reader = p

fmt.Println(r == nil)   // false!
// r holds (type=*File, value=nil) — the interface is not nil because it has a type
```

An interface is nil only when both its type and value are nil. This is a common source of bugs when returning nil pointers through interfaces.

```go
// BUG
func getReader() io.Reader {
    var f *File = nil
    return f  // returns non-nil interface with nil value
}

// FIX
func getReader() io.Reader {
    return nil  // returns actual nil interface
}
```

---

## Key Takeaways

- Interfaces are satisfied implicitly. No `implements` keyword — just have the methods.
- Keep interfaces small (1-3 methods). `io.Reader` has one method and is the most powerful interface in Go.
- Define interfaces where they are consumed, not where they are implemented.
- Accept interfaces, return concrete types.
- The empty interface (`any`) accepts everything but discards type safety. Prefer specific interfaces or generics.
- An interface wrapping a nil pointer is NOT nil. This is the most common interface-related bug.
