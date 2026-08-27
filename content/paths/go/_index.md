---
title: "Go"
weight: 75
bookCollapseSection: true
---

# Go

A pragmatic, opinionated language designed for building reliable, efficient software at scale — from CLI tools and microservices to infrastructure systems and cloud platforms.

## Overview

Go (Golang) was created at Google in 2009 by Robert Griesemer, Rob Pike, and Ken Thompson to address the pain of building large-scale networked services in C++ and Java. It prioritises simplicity, fast compilation, built-in concurrency, and a rich standard library that covers most needs without third-party dependencies.

Go is the language behind Docker, Kubernetes, Terraform, Prometheus, and much of the cloud-native ecosystem. It compiles to a single static binary with no runtime dependencies, starts instantly, and cross-compiles trivially. If you work in cloud infrastructure, platform engineering, or backend services, Go is a first-class tool.

This path assumes you can already program in at least one language. It focuses on what makes Go different — not on explaining what a variable is.

## Prerequisites

- Comfortable programming in any language (Python, Java, JavaScript, etc.)
- Basic command-line proficiency
- Familiarity with HTTP and JSON (for the standard library sections)

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Go Fundamentals]({{< relref "01-fundamentals" >}}) | Philosophy, toolchain, packages, modules, hello world |
| 2 | [Types & Variables]({{< relref "02-types-and-variables" >}}) | Basic types, zero values, constants, iota, strings and runes |
| 3 | [Control Flow]({{< relref "03-control-flow" >}}) | if, for, switch, defer, range |
| 4 | [Functions]({{< relref "04-functions" >}}) | Multiple returns, variadic, closures, first-class functions |
| 5 | [Structs & Methods]({{< relref "05-structs-and-methods" >}}) | Struct types, embedding, methods, receiver types, tags |
| 6 | [Interfaces]({{< relref "06-interfaces" >}}) | Implicit satisfaction, type assertions, key interfaces |
| 7 | [Concurrency]({{< relref "07-concurrency" >}}) | Goroutines, channels, select, sync primitives, patterns |
| 8 | [Error Handling]({{< relref "08-error-handling" >}}) | error interface, wrapping, sentinel errors, panic/recover |
| 9 | [Generics]({{< relref "09-generics" >}}) | Type parameters, constraints, generic functions and types |
| 10 | [Standard Library]({{< relref "10-standard-library" >}}) | net/http, encoding/json, context, testing, io, slog |
| 11 | [Modules & Tooling]({{< relref "11-modules-and-tooling" >}}) | go.mod, testing, linting, profiling, cross-compilation |
| 12 | [Patterns & Idioms]({{< relref "12-patterns-and-idioms" >}}) | Table-driven tests, functional options, project layout, Docker builds |

## What Makes Go Different

| Feature | Go's Approach | Contrast |
|---------|--------------|----------|
| Concurrency | Goroutines + channels (CSP model) | Threads + locks (Java, C++) |
| Error handling | Explicit returns, no exceptions | try/catch (Java, Python) |
| Inheritance | Composition via embedding, no classes | Class hierarchies |
| Generics | Added in 1.18, deliberately simple | Full type systems (Rust, Haskell) |
| Dependencies | Single binary, no runtime | JVM, Node runtime, Python interpreter |
| Formatting | `gofmt` — one true style, enforced | Debates over style guides |
| Unused imports | Compile error | Warning at best |

## Go Version

This path targets **Go 1.22+** (2024). Key version milestones:

| Version | Notable Additions |
|---------|------------------|
| 1.11 | Modules (go.mod) |
| 1.13 | Error wrapping (fmt.Errorf %w) |
| 1.16 | embed package, io/fs |
| 1.18 | Generics, fuzzing |
| 1.21 | log/slog, slices/maps packages |
| 1.22 | Range over integers, enhanced routing |
