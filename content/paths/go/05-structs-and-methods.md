---
title: "Structs & Methods"
weight: 5
---

# Structs & Methods

Go has no classes. Structs are the primary way to define custom data types, and methods are functions with a receiver parameter that give structs behaviour.

---

## Defining Structs

```go
type User struct {
    ID        int
    FirstName string
    LastName  string
    Email     string
    CreatedAt time.Time
}
```

### Creating Instances

```go
// Named fields (preferred — order-independent, self-documenting)
u := User{
    ID:        1,
    FirstName: "Alice",
    LastName:  "Chen",
    Email:     "alice@example.com",
    CreatedAt: time.Now(),
}

// Positional (fragile — breaks if fields are reordered)
u := User{1, "Alice", "Chen", "alice@example.com", time.Now()}

// Zero value
var u User  // all fields are zero values

// Pointer
u := &User{ID: 1, FirstName: "Alice"}
```

### Accessing Fields

```go
fmt.Println(u.FirstName)
u.Email = "new@example.com"

// Pointer access — Go auto-dereferences
p := &u
fmt.Println(p.FirstName)  // same as (*p).FirstName
```

---

## Methods

A method is a function with a **receiver** — it binds the function to a type:

```go
func (u User) FullName() string {
    return u.FirstName + " " + u.LastName
}

// Call
user := User{FirstName: "Alice", LastName: "Chen"}
fmt.Println(user.FullName())  // "Alice Chen"
```

### Value Receivers vs Pointer Receivers

```go
// Value receiver — operates on a COPY
func (u User) FullName() string {
    return u.FirstName + " " + u.LastName
}

// Pointer receiver — operates on the ORIGINAL
func (u *User) SetEmail(email string) {
    u.Email = email  // modifies the original
}
```

### When to Use Which

| Use Pointer Receiver | Use Value Receiver |
|---------------------|-------------------|
| Method modifies the receiver | Method only reads the receiver |
| Struct is large (avoid copying) | Struct is small (few fields) |
| Consistency — if any method uses pointer, all should | Immutable semantics desired |
| The type is meant to be mutated | The type represents a value (time.Time, color.RGBA) |

**Rule of thumb:** If in doubt, use a pointer receiver. Consistency matters more than micro-optimisation.

### Receiver Naming

Use a 1-2 letter abbreviation of the type, not `this` or `self`:

```go
func (u *User) Validate() error { ... }
func (s *Server) Start() error { ... }
func (r *Repository) FindByID(id int) (*User, error) { ... }
```

---

## Constructor Pattern

Go has no constructors. Use a `NewXxx` function:

```go
func NewUser(firstName, lastName, email string) (*User, error) {
    if email == "" {
        return nil, fmt.Errorf("email is required")
    }
    return &User{
        ID:        generateID(),
        FirstName: firstName,
        LastName:  lastName,
        Email:     email,
        CreatedAt: time.Now(),
    }, nil
}
```

Return a pointer when:
- The struct is large
- The caller needs to modify it
- Nil indicates absence

Return a value when:
- The struct is small and immutable
- The zero value is useful

---

## Struct Embedding (Composition)

Go uses composition instead of inheritance. Embed one struct inside another:

```go
type Address struct {
    Street  string
    City    string
    Country string
}

type Employee struct {
    User             // embedded — fields and methods are promoted
    Address          // embedded
    Department string
    Salary     float64
}

e := Employee{
    User:       User{FirstName: "Alice", LastName: "Chen"},
    Address:    Address{City: "Barcelona", Country: "Spain"},
    Department: "Engineering",
    Salary:     95000,
}

// Promoted fields — accessed directly
fmt.Println(e.FirstName)  // from User
fmt.Println(e.City)       // from Address
fmt.Println(e.FullName()) // method from User
```

### Embedding Is Not Inheritance

| Embedding (Go) | Inheritance (Java/C++) |
|----------------|----------------------|
| Composition — "has a" | Hierarchy — "is a" |
| No polymorphism through embedding | Virtual dispatch |
| Promoted methods can be overridden | Overriding with `super` |
| Explicit — you see the embedded type | Implicit via class chain |

```go
// "Override" a promoted method
func (e Employee) FullName() string {
    return fmt.Sprintf("%s (%s)", e.User.FullName(), e.Department)
}
```

---

## Struct Tags

Tags are metadata strings attached to struct fields, read at runtime via reflection:

```go
type User struct {
    ID        int    `json:"id" db:"user_id"`
    FirstName string `json:"first_name" db:"first_name"`
    LastName  string `json:"last_name" db:"last_name"`
    Email     string `json:"email" db:"email" validate:"required,email"`
    Password  string `json:"-" db:"password_hash"`  // "-" = omit from JSON
}
```

### Common Tag Formats

| Library | Tag | Purpose |
|---------|-----|---------|
| encoding/json | `json:"name,omitempty"` | JSON serialisation |
| encoding/xml | `xml:"name,attr"` | XML serialisation |
| database/sql | `db:"column_name"` | Database mapping (sqlx) |
| validator | `validate:"required,min=3"` | Struct validation |
| yaml | `yaml:"name"` | YAML serialisation |
| mapstructure | `mapstructure:"name"` | Config file mapping |

### JSON Struct Tags in Detail

```go
type Response struct {
    Status  int    `json:"status"`            // rename field
    Message string `json:"message,omitempty"` // omit if empty
    Data    any    `json:"data"`              // any type
    Secret  string `json:"-"`                 // never serialise
}
```

---

## Anonymous Structs

Useful for one-off structures (test data, JSON responses, config):

```go
// Inline struct
point := struct {
    X, Y int
}{X: 10, Y: 20}

// Table-driven test data
tests := []struct {
    name     string
    input    string
    expected int
}{
    {"empty", "", 0},
    {"single", "a", 1},
    {"multi", "abc", 3},
}
```

---

## Comparing Structs

Structs are comparable if all their fields are comparable:

```go
a := User{ID: 1, FirstName: "Alice"}
b := User{ID: 1, FirstName: "Alice"}
fmt.Println(a == b)  // true

// Structs with slices or maps are NOT comparable with ==
// Use reflect.DeepEqual or write a custom Equal method
```

---

## Key Takeaways

- Structs are Go's primary data type. No classes, no constructors — use `NewXxx` factory functions.
- Methods are functions with a receiver. Use pointer receivers for mutation, value receivers for read-only.
- Embedding promotes fields and methods — composition over inheritance. It is NOT inheritance.
- Struct tags are metadata for serialisation libraries (JSON, database, validation).
- Use named fields in struct literals for clarity and resilience to field reordering.
- The zero value of a well-designed struct should be usable. Design accordingly.
