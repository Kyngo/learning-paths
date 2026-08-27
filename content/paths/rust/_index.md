---
title: "Rust"
weight: 76
bookCollapseSection: true
---

# Rust

A systems programming language that guarantees memory safety and thread safety at compile time — without a garbage collector.

## Overview

Rust was created by Graydon Hoare at Mozilla and released in 2015. It solves a problem that plagued systems programming for decades: how to write fast, low-level code without memory bugs (use-after-free, buffer overflows, data races) that are the root cause of ~70% of security vulnerabilities in C/C++ codebases (per Microsoft and Google studies).

Rust achieves this through its **ownership system** — a set of compile-time rules that ensure memory is always valid, always freed exactly once, and never accessed concurrently without synchronisation. If your code compiles, an entire class of bugs is eliminated.

Rust is used in Firefox, the Linux kernel, Android, AWS (Firecracker, Bottlerocket), Cloudflare, Discord, Figma, and the foundations of many CLI tools (ripgrep, fd, bat, exa). It has been voted "most admired language" on Stack Overflow for years running.

This path assumes you can already program and understand concepts like pointers, stack vs heap, and concurrency. It focuses on what makes Rust unique.

## Prerequisites

- Comfortable programming in at least one language (C, C++, Go, Java, or similar)
- Understanding of stack vs heap memory allocation
- Basic familiarity with command-line tools
- Willingness to fight the compiler — it will reject code you think is correct, and it will be right

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Getting Started]({{< relref "01-getting-started" >}}) | Installation, cargo, project structure, hello world |
| 2 | [Types & Variables]({{< relref "02-types-and-variables" >}}) | Primitives, mutability, shadowing, tuples, arrays, type inference |
| 3 | [Ownership & Borrowing]({{< relref "03-ownership-and-borrowing" >}}) | The ownership model, moves, borrows, lifetimes, the borrow checker |
| 4 | [Structs & Enums]({{< relref "04-structs-and-enums" >}}) | Structs, methods, enums, pattern matching, Option, Result |
| 5 | [Error Handling]({{< relref "05-error-handling" >}}) | Result, Option, the ? operator, custom errors, anyhow/thiserror |
| 6 | [Collections & Iterators]({{< relref "06-collections-and-iterators" >}}) | Vec, HashMap, iterators, closures, iterator adaptors, collect |
| 7 | [Traits & Generics]({{< relref "07-traits-and-generics" >}}) | Traits, trait bounds, generics, associated types, trait objects |
| 8 | [Lifetimes]({{< relref "08-lifetimes" >}}) | Lifetime annotations, elision rules, structs with references, 'static |
| 9 | [Concurrency]({{< relref "09-concurrency" >}}) | Threads, Send/Sync, Mutex, Arc, channels, async/await, tokio |
| 10 | [Smart Pointers & Memory]({{< relref "10-smart-pointers" >}}) | Box, Rc, Arc, RefCell, Cow, interior mutability, unsafe |
| 11 | [Modules & Cargo]({{< relref "11-modules-and-cargo" >}}) | Module system, crates, workspaces, features, testing, publishing |
| 12 | [Patterns & Ecosystem]({{< relref "12-patterns-and-ecosystem" >}}) | Builder, newtype, typestate, serde, clap, tokio, key crates |

## What Makes Rust Different

| Feature | Rust's Approach | Contrast |
|---------|---------------|----------|
| Memory safety | Ownership + borrow checker (compile-time) | GC (Java, Go, Python) or manual (C, C++) |
| Null safety | No null — uses `Option<T>` | Null references everywhere else |
| Error handling | `Result<T, E>` — no exceptions | try/catch (Java), panic (Go) |
| Concurrency safety | Send/Sync traits — data races impossible | Shared mutable state (C++, Java) |
| Zero-cost abstractions | Iterators, generics compile to optimal machine code | Virtual dispatch overhead (Java) |
| No runtime | No GC, no VM, no interpreter | JVM, Node, Python interpreter |
| Package manager | Cargo — build, test, bench, publish | Make + manual deps (C), go mod (Go) |

## Rust Edition

This path targets **Rust 2021 edition** (stable since Rust 1.56). Key edition milestones:

| Edition | Notable Changes |
|---------|----------------|
| 2015 | Initial stable release |
| 2018 | Module system overhaul, async/await (1.39), NLL borrow checker |
| 2021 | Disjoint capture in closures, IntoIterator for arrays, cleaner defaults |
| 2024 | (In progress) RPIT lifetime capture rules, `gen` blocks |
