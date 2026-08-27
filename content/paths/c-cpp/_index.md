---
title: "C / C++"
weight: 77
bookCollapseSection: true
---

# C / C++

The foundational systems languages — C for operating systems, embedded, and kernels; C++ for performance-critical applications, game engines, and infrastructure. Understanding them gives you direct access to how computers actually work.

## Overview

C (1972) and C++ (1985) are distinct languages that share a common heritage. C is minimal by design — it maps closely to hardware and gives you manual control over every byte. C++ extends C with classes, templates, RAII, and a massive standard library while maintaining zero-cost abstraction as a core principle.

Together they power: the Linux kernel, Windows, macOS, PostgreSQL, SQLite, Redis, Nginx, Chrome (V8), Firefox, the JVM (HotSpot), CPython, LLVM/Clang, Unreal Engine, most game engines, and virtually every embedded system.

This path teaches both languages together because understanding C is essential for understanding C++, and many real-world codebases mix them. The early sections focus on C fundamentals; later sections introduce C++ features progressively.

## Prerequisites

- Comfortable programming in any language
- Basic understanding of how programs execute (stack, heap, function calls)
- Command-line proficiency
- Willingness to manage memory manually — the compiler will not save you

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [C Fundamentals]({{< relref "01-c-fundamentals" >}}) | Compilation, types, printf, control flow, functions |
| 2 | [Pointers & Memory]({{< relref "02-pointers-and-memory" >}}) | Pointers, pointer arithmetic, arrays, stack vs heap, malloc/free |
| 3 | [Strings & Arrays]({{< relref "03-strings-and-arrays" >}}) | C strings, string.h, multidimensional arrays, buffer overflows |
| 4 | [Structs & Unions]({{< relref "04-structs-and-unions" >}}) | Struct layout, unions, bitfields, typedef, enums, memory alignment |
| 5 | [The Preprocessor & Compilation]({{< relref "05-preprocessor-and-compilation" >}}) | #include, #define, macros, header guards, compilation model, linking |
| 6 | [File I/O & System Calls]({{< relref "06-file-io-and-system-calls" >}}) | stdio, file descriptors, POSIX I/O, errno, signals |
| 7 | [C++ Core]({{< relref "07-cpp-core" >}}) | Classes, constructors/destructors, RAII, references, namespaces, const |
| 8 | [C++ Object-Oriented]({{< relref "08-cpp-oop" >}}) | Inheritance, virtual functions, polymorphism, abstract classes, operator overloading |
| 9 | [Templates & STL]({{< relref "09-templates-and-stl" >}}) | Function/class templates, vector, map, algorithms, iterators |
| 10 | [Modern C++]({{< relref "10-modern-cpp" >}}) | Smart pointers, move semantics, lambdas, auto, structured bindings, std::optional |
| 11 | [Concurrency]({{< relref "11-concurrency" >}}) | pthreads, std::thread, mutex, atomics, condition variables, async |
| 12 | [Build Systems & Tooling]({{< relref "12-build-systems-and-tooling" >}}) | Make, CMake, sanitizers, Valgrind, GDB, static analysis, packaging |

## C vs C++ at a Glance

| Feature | C | C++ |
|---------|---|-----|
| Paradigm | Procedural | Multi-paradigm (procedural, OOP, generic, functional) |
| Memory management | Manual (malloc/free) | Manual + RAII + smart pointers |
| Strings | `char*` + null terminator | `std::string` |
| Generics | Macros or `void*` | Templates |
| Error handling | Return codes + errno | Exceptions + std::optional + std::expected |
| Standard library | Minimal (stdio, stdlib, string) | Massive (containers, algorithms, I/O, threading) |
| ABI stability | Stable (de facto) | Fragile (name mangling, vtable layout) |
| Use case | Kernels, embedded, drivers | Applications, engines, infrastructure |

## Standard Versions

| Language | Standard | Key Additions |
|----------|----------|---------------|
| C89/C90 | Original ANSI C | The baseline |
| C99 | `//` comments, `bool`, variable-length arrays, `stdint.h` | |
| C11 | `_Atomic`, `_Thread_local`, `static_assert` | |
| C17 | Bug fixes only | |
| C23 | `#embed`, `typeof`, `nullptr`, digit separators | |
| C++11 | `auto`, lambdas, move semantics, smart pointers, threads | The modern C++ revolution |
| C++14 | Generic lambdas, relaxed constexpr | |
| C++17 | `std::optional`, structured bindings, `if constexpr`, `std::filesystem` | |
| C++20 | Concepts, ranges, coroutines, modules, `std::format` | |
| C++23 | `std::expected`, `std::print`, `std::mdspan` | |

This path targets **C17** and **C++17** as the baseline, with notes on C++20/23 features where relevant.
