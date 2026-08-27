---
title: "Patterns & Idioms"
weight: 12
---

# Patterns & Idioms

Go has a strong culture of idiomatic code. These patterns are not in the language specification but are universally understood and expected by the Go community.

---

## Project Layout

### Standard Layout (for Libraries and Services)

```
myservice/
├── cmd/
│   └── myservice/
│       └── main.go           # entry point
├── internal/
│   ├── handler/              # HTTP handlers
│   │   └── handler.go
│   ├── service/              # business logic
│   │   └── service.go
│   ├── repository/           # data access
│   │   └── repository.go
│   └── model/                # domain types
│       └── model.go
├── pkg/                      # importable by other modules (optional)
│   └── client/
│       └── client.go
├── api/                      # API specs (OpenAPI, protobuf)
│   └── openapi.yaml
├── migrations/               # database migrations
│   └── 001_create_users.sql
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── README.md
```

### Small Tool / CLI

```
mytool/
├── main.go
├── go.mod
└── go.sum
```

### Key Directory Rules

| Directory | Visibility | Purpose |
|-----------|-----------|---------|
| `cmd/` | — | Each subdirectory is a separate binary |
| `internal/` | Module-private | Cannot be imported by other modules |
| `pkg/` | Public | Can be imported — use only when you intend external use |
| No `/src` | — | Go does not use a `src/` directory (unlike Java) |

---

## Table-Driven Tests

The canonical Go testing pattern:

```go
func TestParseSize(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int64
        wantErr bool
    }{
        {"bytes", "100B", 100, false},
        {"kilobytes", "10KB", 10240, false},
        {"megabytes", "5MB", 5242880, false},
        {"gigabytes", "2GB", 2147483648, false},
        {"no unit", "100", 0, true},
        {"empty", "", 0, true},
        {"negative", "-5MB", 0, true},
        {"zero", "0B", 0, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseSize(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseSize(%q) error = %v, wantErr %v", tt.input, err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("ParseSize(%q) = %d, want %d", tt.input, got, tt.want)
            }
        })
    }
}
```

### Why Table-Driven?

- Adding a test case is one line
- Each case is named and runs as a subtest
- Test logic is written once
- Easy to generate test cases
- `go test -run TestParseSize/kilobytes` runs a single case

---

## Functional Options

The pattern for constructors with many optional parameters:

```go
type Server struct {
    addr         string
    port         int
    readTimeout  time.Duration
    writeTimeout time.Duration
    logger       *slog.Logger
}

type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithReadTimeout(d time.Duration) Option {
    return func(s *Server) { s.readTimeout = d }
}

func WithWriteTimeout(d time.Duration) Option {
    return func(s *Server) { s.writeTimeout = d }
}

func WithLogger(l *slog.Logger) Option {
    return func(s *Server) { s.logger = l }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:         addr,
        port:         8080,
        readTimeout:  5 * time.Second,
        writeTimeout: 10 * time.Second,
        logger:       slog.Default(),
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Clean, self-documenting usage
server := NewServer("0.0.0.0",
    WithPort(9090),
    WithReadTimeout(30*time.Second),
    WithLogger(customLogger),
)
```

### Alternatives

| Pattern | Pros | Cons |
|---------|------|------|
| Functional options | Extensible, self-documenting, zero values work | More boilerplate |
| Config struct | Simple, familiar | Must distinguish zero from "not set" |
| Builder | Chain calls | Mutable, error-prone |

---

## Dependency Injection

Go uses interfaces and constructor injection — no DI framework needed:

```go
// Define the dependency as an interface (in the consumer package)
type UserRepository interface {
    FindByID(ctx context.Context, id int) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Service depends on the interface
type UserService struct {
    repo   UserRepository
    logger *slog.Logger
}

func NewUserService(repo UserRepository, logger *slog.Logger) *UserService {
    return &UserService{repo: repo, logger: logger}
}

// Wire it up in main
func main() {
    db := connectDB()
    repo := postgres.NewUserRepository(db)
    logger := slog.Default()
    service := NewUserService(repo, logger)
    handler := NewHandler(service)
    // ...
}
```

### Test with a Mock

```go
type mockRepo struct {
    users map[int]*User
}

func (m *mockRepo) FindByID(ctx context.Context, id int) (*User, error) {
    u, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}

func TestGetUser(t *testing.T) {
    repo := &mockRepo{users: map[int]*User{1: {Name: "Alice"}}}
    svc := NewUserService(repo, slog.Default())

    user, err := svc.GetUser(context.Background(), 1)
    if err != nil {
        t.Fatal(err)
    }
    if user.Name != "Alice" {
        t.Errorf("got %q, want Alice", user.Name)
    }
}
```

---

## Graceful Shutdown

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    srv := &http.Server{Addr: ":8080", Handler: mux}

    // Start server
    go func() {
        slog.Info("server starting", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            slog.Error("server error", "error", err)
            os.Exit(1)
        }
    }()

    // Wait for signal
    <-ctx.Done()
    slog.Info("shutting down...")

    // Give in-flight requests time to finish
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown error", "error", err)
    }
    slog.Info("server stopped")
}
```

---

## Middleware Pattern

```go
type Middleware func(http.Handler) http.Handler

func Logging(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            next.ServeHTTP(w, r)
            logger.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "duration", time.Since(start),
            )
        })
    }
}

func Recovery() Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if err := recover(); err != nil {
                    http.Error(w, "internal error", 500)
                    slog.Error("panic recovered", "error", err)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}

// Chain middleware
func Chain(h http.Handler, mw ...Middleware) http.Handler {
    for i := len(mw) - 1; i >= 0; i-- {
        h = mw[i](h)
    }
    return h
}

// Usage
handler := Chain(mux, Logging(logger), Recovery())
```

---

## Makefile

```makefile
.PHONY: build test lint run clean

APP_NAME := myservice
VERSION  := $(shell git describe --tags --always --dirty)
LDFLAGS  := -X main.version=$(VERSION) -X main.commit=$(shell git rev-parse --short HEAD)

build:
	CGO_ENABLED=0 go build -ldflags "$(LDFLAGS)" -o bin/$(APP_NAME) ./cmd/$(APP_NAME)

test:
	go test -race -count=1 ./...

lint:
	golangci-lint run ./...

run: build
	./bin/$(APP_NAME)

coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

clean:
	rm -rf bin/ coverage.out coverage.html
```

---

## Docker Multi-Stage Build

```dockerfile
# Build stage
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# Runtime stage
FROM gcr.io/distroless/static-debian12
COPY --from=build /app/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

| Stage | Base Image | Size |
|-------|-----------|------|
| Build | `golang:1.22-alpine` | ~300 MB |
| Runtime | `distroless/static` | ~2 MB |
| Final image | — | ~10-15 MB (binary + distroless) |

### Why Distroless?

- No shell, no package manager, no unnecessary binaries
- Smaller attack surface (no `sh`, no `curl`)
- Meets security scanning requirements

### Alternative: `scratch`

```dockerfile
FROM scratch
COPY --from=build /app/server /server
ENTRYPOINT ["/server"]
```

Even smaller (~5 MB) but has no TLS certificates, no timezone data, no `/etc/passwd`. You must embed certificates or copy `/etc/ssl/certs` from the build stage.

---

## Key Takeaways

- Project layout: `cmd/` for entry points, `internal/` for private code, keep it flat until complexity demands structure.
- Table-driven tests are the standard pattern. Each test case is a named subtest.
- Functional options (`WithXxx`) are the idiomatic way to build flexible constructors.
- Dependency injection in Go is manual — pass interfaces through constructors, no framework needed.
- Graceful shutdown: `signal.NotifyContext` + `server.Shutdown` with a timeout.
- Multi-stage Docker builds with distroless produce tiny (~10 MB), secure images from a single static binary.
- A Makefile ties everything together: build, test, lint, run, clean.
