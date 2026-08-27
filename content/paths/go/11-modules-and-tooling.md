---
title: "Modules & Tooling"
weight: 11
---

# Modules & Tooling

Go ships with a comprehensive toolchain — dependency management, testing, linting, profiling, and cross-compilation are all built in or a single `go install` away.

---

## Modules

### go.mod

```
module github.com/yourname/myproject

go 1.22

require (
    github.com/gorilla/mux v1.8.1
    github.com/lib/pq v1.10.9
)

require (
    // indirect dependencies
    golang.org/x/text v0.14.0 // indirect
)
```

### Common Module Commands

```bash
go mod init github.com/you/project  # create go.mod
go mod tidy                          # add missing, remove unused
go mod download                      # download dependencies
go mod verify                        # verify checksums
go mod vendor                        # copy deps into ./vendor
go mod graph                         # show dependency graph
go get github.com/pkg@v1.2.3        # add/update specific version
go get -u ./...                      # update all direct deps
```

### Version Selection

Go uses **Minimum Version Selection (MVS)**: if A requires lib@v1.2 and B requires lib@v1.3, Go picks v1.3 (the minimum version that satisfies all requirements). No SAT solver, deterministic, reproducible.

### Replace and Exclude

```go
// go.mod

// Use a local copy during development
replace github.com/myorg/lib => ../lib

// Use a fork
replace github.com/original/lib => github.com/myfork/lib v1.0.0

// Exclude a broken version
exclude github.com/pkg/lib v1.2.3
```

### Private Modules

```bash
# Tell Go to fetch from private Git (not proxy)
export GONOSUMCHECK=github.com/myorg/*
export GOPRIVATE=github.com/myorg/*
export GONOSUMDB=github.com/myorg/*
```

---

## Testing

### Test File Conventions

```
user.go       → user_test.go      (same package)
user.go       → user_test.go      (package user_test — black-box testing)
```

### Running Tests

```bash
go test ./...                # all packages
go test -v ./...             # verbose
go test -run TestCreate      # pattern match
go test -short ./...         # skip long tests
go test -count=1 ./...       # disable caching
go test -parallel 4 ./...    # parallel test limit
```

### Test Helpers

```go
func TestSomething(t *testing.T) {
    t.Helper()  // marks this as a helper — errors show caller's line
    // ...
}
```

### Subtests

```go
func TestMath(t *testing.T) {
    t.Run("addition", func(t *testing.T) {
        if add(1, 2) != 3 {
            t.Error("1+2 should be 3")
        }
    })
    t.Run("subtraction", func(t *testing.T) {
        if sub(5, 3) != 2 {
            t.Error("5-3 should be 2")
        }
    })
}
```

### Test Coverage

```bash
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out  # open in browser
go tool cover -func=coverage.out  # per-function breakdown
```

### Benchmarks

```go
func BenchmarkSort(b *testing.B) {
    data := generateData(1000)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sort.Ints(data)
    }
}
```

```bash
go test -bench=. -benchmem ./...
# BenchmarkSort-10    50000    24350 ns/op    8192 B/op    1 allocs/op
```

### Fuzzing (Go 1.18+)

```go
func FuzzReverse(f *testing.F) {
    f.Add("hello")
    f.Add("世界")
    f.Fuzz(func(t *testing.T, s string) {
        rev := Reverse(s)
        doubleRev := Reverse(rev)
        if s != doubleRev {
            t.Errorf("double reverse of %q got %q", s, doubleRev)
        }
    })
}
```

```bash
go test -fuzz=FuzzReverse -fuzztime=30s
```

---

## Linting and Static Analysis

### go vet (Built-in)

```bash
go vet ./...
```

Catches: printf format mismatches, unreachable code, suspicious constructs, incorrect struct tags.

### golangci-lint (Community Standard)

```bash
# Install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Run
golangci-lint run ./...
```

Runs 100+ linters including: `errcheck` (unchecked errors), `staticcheck` (advanced checks), `gosimple` (simplifications), `govet`, `revive`, `ineffassign`.

### Configuration (.golangci.yml)

```yaml
linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - revive
    - gosec
    - ineffassign
    - gocritic
  disable:
    - depguard

linters-settings:
  revive:
    rules:
      - name: exported
        severity: warning

run:
  timeout: 5m
```

---

## Profiling

### CPU Profile

```go
import "runtime/pprof"

f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
// ... code to profile ...
```

### HTTP Profiling (Production-Ready)

```go
import _ "net/http/pprof"

go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Memory profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Interactive commands
(pprof) top 20
(pprof) web          # opens browser visualisation
(pprof) list funcName  # source-annotated profile
```

### Memory Profiling

```go
func BenchmarkAllocations(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        process()
    }
}
```

### Execution Tracing

```bash
go test -trace=trace.out ./...
go tool trace trace.out  # opens interactive browser view
```

Shows goroutine scheduling, GC events, system calls, and network I/O on a timeline.

---

## Build

### Build Tags

```go
//go:build linux
// +build linux

package mypackage
// This file only compiles on Linux
```

```bash
go build -tags "integration,debug" ./...
```

### Embedding Files

```go
import "embed"

//go:embed static/*
var staticFiles embed.FS

//go:embed version.txt
var version string

//go:embed config.json
var configData []byte
```

### Linker Flags (Injecting Build Info)

```bash
go build -ldflags "-X main.version=1.2.3 -X main.commit=$(git rev-parse --short HEAD)"
```

```go
var version = "dev"
var commit = "unknown"
```

### Cross-Compilation

```bash
GOOS=linux GOARCH=amd64 go build -o app-linux
GOOS=darwin GOARCH=arm64 go build -o app-mac
GOOS=windows GOARCH=amd64 go build -o app.exe

# CGO disabled (for fully static binaries)
CGO_ENABLED=0 GOOS=linux go build -o app
```

---

## Code Generation

```go
//go:generate stringer -type=Status
//go:generate mockgen -source=repo.go -destination=mock_repo.go
```

```bash
go generate ./...
```

Common generators: `stringer` (enum String methods), `mockgen` (mocks for testing), `protoc-gen-go` (protobuf), `sqlc` (type-safe SQL).

---

## Key Takeaways

- `go mod tidy` is the most important module command — run it after any dependency change.
- Go uses Minimum Version Selection — deterministic, no lock file ambiguity.
- Table-driven tests with `t.Run` subtests are the standard testing pattern. Always run with `-race`.
- `golangci-lint` is the community-standard linter — configure it once and run in CI.
- `pprof` is built in and production-ready. Use it for CPU, memory, and goroutine profiling.
- `CGO_ENABLED=0` gives fully static binaries. `GOOS`/`GOARCH` for cross-compilation.
- `//go:embed` bundles files into binaries — no more asset loading at runtime.
