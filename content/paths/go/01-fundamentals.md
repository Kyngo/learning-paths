---
title: "Go Fundamentals"
weight: 1
---

# Go Fundamentals

Go is deliberately simple. The language specification fits in a few dozen pages. There are no classes, no inheritance, no exceptions, no operator overloading, and no implicit type conversions. This is not a limitation — it is the design.

---

## Philosophy

Go was born from frustration with the compile times, complexity, and dependency management of C++ and Java at Google scale. Its design principles:

| Principle | Manifestation |
|-----------|--------------|
| Simplicity | 25 keywords, one loop construct, no ternary operator |
| Readability | `gofmt` enforces a single style — no formatting debates |
| Fast compilation | Compiles a million lines of code in seconds |
| Explicit over implicit | No hidden constructors, no magic methods, no implicit conversions |
| Composition over inheritance | Struct embedding instead of class hierarchies |
| Concurrency as a first-class concept | Goroutines and channels built into the language |
| Batteries included | Standard library covers HTTP, JSON, crypto, testing, profiling |

### The Go Proverbs

From Rob Pike's 2015 talk, distilled wisdom:

- "Don't communicate by sharing memory; share memory by communicating."
- "Concurrency is not parallelism."
- "Channels orchestrate; mutexes serialize."
- "The bigger the interface, the weaker the abstraction."
- "Make the zero value useful."
- "A little copying is better than a little dependency."
- "Clear is better than clever."
- "Errors are values."
- "Don't just check errors, handle them gracefully."

---

## Installing Go

```bash
# macOS
brew install go

# Linux (download from go.dev)
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Verify
go version
# go version go1.22.5 darwin/arm64
```

### Environment

```bash
go env GOPATH     # ~/go (default workspace)
go env GOROOT     # /usr/local/go (Go installation)
go env GOOS       # darwin, linux, windows
go env GOARCH     # amd64, arm64
```

---

## Hello World

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

Every Go program starts in `package main` with a `func main()`. No boilerplate beyond this.

### Running and Building

```bash
# Run directly (compiles and executes)
go run main.go

# Build a binary
go build -o myapp main.go
./myapp

# Install to $GOPATH/bin
go install
```

---

## Modules

Go modules are the dependency management system (since Go 1.11). A module is defined by a `go.mod` file at the root of your project.

### Creating a Module

```bash
mkdir myproject && cd myproject
go mod init github.com/yourname/myproject
```

This creates `go.mod`:

```
module github.com/yourname/myproject

go 1.22
```

### Adding Dependencies

```bash
# Add a dependency (fetches and records it)
go get github.com/gorilla/mux@latest

# Tidy — remove unused, add missing
go mod tidy

# Verify checksums
go mod verify
```

Dependencies are recorded in `go.mod` and their checksums in `go.sum`. Both files should be committed to version control.

### Module Structure

```
myproject/
├── go.mod
├── go.sum
├── main.go
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── handler.go
│   └── service/
│       └── service.go
└── pkg/
    └── util/
        └── util.go
```

| Directory | Purpose |
|-----------|---------|
| `cmd/` | Entry points (each subdirectory builds a separate binary) |
| `internal/` | Private packages — cannot be imported by other modules |
| `pkg/` | Public packages — importable by external modules |

---

## Packages

Every `.go` file begins with a `package` declaration. All files in the same directory must belong to the same package.

### Visibility

Go uses capitalisation for visibility — no `public`/`private` keywords:

```go
package user

// Exported (uppercase first letter) — visible outside the package
func CreateUser(name string) User { ... }
type User struct { ... }
const MaxRetries = 3

// Unexported (lowercase first letter) — package-private
func validateEmail(email string) bool { ... }
type config struct { ... }
const defaultTimeout = 30
```

### Importing Packages

```go
import (
    "fmt"                              // standard library
    "net/http"                         // standard library (nested)
    "github.com/gorilla/mux"           // third-party
    "github.com/yourname/myproject/internal/handler"  // local
)
```

**Unused imports are a compile error.** Use `_` to import for side effects only:

```go
import _ "github.com/lib/pq"  // registers PostgreSQL driver
```

### The `init()` Function

Each package can define one or more `init()` functions that run automatically when the package is imported:

```go
package config

var settings map[string]string

func init() {
    settings = loadFromEnvironment()
}
```

`init()` runs before `main()`. Use it sparingly — implicit initialisation makes code harder to reason about.

---

## Formatting

There is exactly one formatting style in Go:

```bash
gofmt -w .     # format all files in place
goimports -w . # format + organise imports
```

`gofmt` is not optional — it is a social contract. Every Go project uses it. Your editor should run it on save.

### Code Style Conventions

| Convention | Example |
|-----------|---------|
| CamelCase for exported names | `HttpHandler`, `ParseJSON` |
| camelCase for unexported names | `httpHandler`, `parseJSON` |
| Acronyms stay uppercase | `HTTPClient`, `XMLParser`, `ID` (not `Id`) |
| Short variable names for small scopes | `i`, `n`, `err`, `ctx`, `r`, `w` |
| Descriptive names for larger scopes | `userRepository`, `connectionPool` |
| Receiver names: 1-2 letter abbreviation | `func (u *User) Name() string` |

---

## The Go Toolchain

```bash
go build      # compile packages and dependencies
go run        # compile and run a program
go test       # run tests
go vet        # report suspicious constructs
go fmt        # format source code (alias for gofmt)
go get        # add/update dependencies
go mod tidy   # clean up go.mod and go.sum
go mod init   # initialise a new module
go install    # compile and install to $GOPATH/bin
go doc        # show documentation
go generate   # run code generation directives
go env        # print Go environment variables
go tool pprof # profiling
go tool trace # execution tracing
```

### Cross-Compilation

Build for any OS/architecture from any machine:

```bash
# Linux AMD64 binary from macOS
GOOS=linux GOARCH=amd64 go build -o myapp-linux

# Windows ARM64
GOOS=windows GOARCH=arm64 go build -o myapp.exe

# All supported platforms
go tool dist list
```

This is a single static binary. No dependencies, no runtime, no container required (though containers are still useful for deployment).

---

## Comments and Documentation

Go uses `//` for line comments and `/* */` for block comments. Documentation is extracted from comments directly preceding declarations:

```go
// Package user provides user management functionality.
package user

// User represents a registered user in the system.
// The zero value is not useful; use NewUser to create instances.
type User struct {
    ID    int
    Name  string
    Email string
}

// NewUser creates a User with the given name and email.
// It returns an error if the email is invalid.
func NewUser(name, email string) (*User, error) {
    // ...
}
```

```bash
# View documentation
go doc github.com/yourname/myproject/user
go doc github.com/yourname/myproject/user.NewUser
```

No JavaDoc-style tags. No `@param`, `@return`, `@throws`. Just plain English sentences starting with the name of the thing being documented.

---

## Key Takeaways

- Go is deliberately simple: 25 keywords, one loop, no exceptions, no inheritance. Simplicity is the feature.
- Every Go program starts in `package main` with `func main()`. Capitalisation controls visibility — no access modifiers.
- Modules (`go.mod`) manage dependencies. `go mod tidy` is your friend.
- `gofmt` is non-negotiable. There is one formatting style and everyone uses it.
- Go compiles to a single static binary that cross-compiles trivially. No runtime, no dependencies.
- The `internal/` directory enforces encapsulation at the module level — code there cannot be imported by external modules.
