---
title: "The Preprocessor & Compilation"
weight: 5
---

# The Preprocessor & Compilation

The C preprocessor is a text substitution engine that runs before compilation. It handles file inclusion, macro expansion, and conditional compilation. Understanding it is essential because it explains header files, include guards, and half the cryptic compiler errors you will encounter.

---

## #include

```c
#include <stdio.h>      // search system include paths
#include "myheader.h"   // search current directory first, then system paths
```

`#include` literally copies the file contents into your source. A header file is just a C file that gets pasted in.

### Header Files

Headers declare interfaces — functions, types, constants — without definitions:

```c
// user.h — declaration
#ifndef USER_H
#define USER_H

typedef struct {
    int id;
    char name[64];
} User;

User *user_create(const char *name);
void user_free(User *u);

#endif // USER_H
```

```c
// user.c — definition (implementation)
#include "user.h"
#include <stdlib.h>
#include <string.h>

User *user_create(const char *name) {
    User *u = malloc(sizeof(User));
    if (!u) return NULL;
    u->id = generate_id();
    strncpy(u->name, name, sizeof(u->name) - 1);
    u->name[sizeof(u->name) - 1] = '\0';
    return u;
}

void user_free(User *u) {
    free(u);
}
```

### Include Guards

Without guards, including a header twice causes "redefinition" errors:

```c
// Traditional guard
#ifndef MY_HEADER_H
#define MY_HEADER_H
// ... declarations ...
#endif

// Modern alternative (widely supported but non-standard)
#pragma once
```

---

## #define — Macros

### Object-Like Macros (Constants)

```c
#define MAX_BUFFER 1024
#define PI 3.14159265358979
#define VERSION "1.2.3"
```

### Function-Like Macros

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))
#define STRINGIFY(x) #x
#define CONCAT(a, b) a##b
```

**Always parenthesise macro arguments and the entire expression:**

```c
// BAD
#define SQUARE(x) x * x
SQUARE(1 + 2)  // expands to: 1 + 2 * 1 + 2 = 5 (not 9!)

// GOOD
#define SQUARE(x) ((x) * (x))
SQUARE(1 + 2)  // expands to: ((1 + 2) * (1 + 2)) = 9
```

### Macro Pitfalls

| Pitfall | Example | Problem |
|---------|---------|---------|
| Double evaluation | `MAX(i++, j++)` | Arguments evaluated twice — side effects doubled |
| Missing parens | `#define MUL(a,b) a*b` | `MUL(1+2, 3)` = `1+2*3` = 7, not 9 |
| Type unsafe | `MAX(int_val, float_val)` | No type checking |
| Hard to debug | — | Macros are expanded before compilation — errors show expanded code |

**Prefer `static inline` functions over function-like macros** in modern C. They are type-safe and debuggable.

---

## Conditional Compilation

```c
#ifdef DEBUG
    printf("debug: x = %d\n", x);
#endif

#ifndef NDEBUG
    assert(x > 0);
#endif

#if defined(_WIN32)
    // Windows-specific code
#elif defined(__linux__)
    // Linux-specific code
#elif defined(__APPLE__)
    // macOS-specific code
#else
    #error "Unsupported platform"
#endif
```

```bash
gcc -DDEBUG -o app main.c    # define DEBUG from command line
```

---

## The Compilation Model

### Separate Compilation

Each `.c` file is compiled independently into an object file (`.o`). The linker combines them:

```bash
gcc -c user.c -o user.o       # compile user.c to object file
gcc -c main.c -o main.o       # compile main.c to object file
gcc user.o main.o -o app      # link into executable
```

### What the Linker Does

1. Resolves symbol references (function calls, global variables)
2. Combines code from multiple object files
3. Links against libraries (libc, libm, etc.)
4. Produces the final executable

### Static vs Dynamic Libraries

| | Static (`.a`) | Dynamic (`.so` / `.dylib`) |
|-|---------------|---------------------------|
| Linking | Copied into executable | Referenced at runtime |
| Binary size | Larger | Smaller |
| Updates | Must recompile | Just replace the library |
| Distribution | Self-contained | Requires library on target |

```bash
# Create static library
ar rcs libuser.a user.o
gcc main.o -L. -luser -o app

# Create shared library
gcc -shared -fPIC -o libuser.so user.o
gcc main.o -L. -luser -o app
```

---

## Compilation Flags Reference

| Flag | Purpose |
|------|---------|
| `-Wall -Wextra -Werror` | Maximum warnings, treat as errors |
| `-std=c17` | C17 standard |
| `-g` | Debug symbols (for GDB) |
| `-O0` / `-O2` / `-O3` / `-Os` | Optimisation (none / moderate / aggressive / size) |
| `-fsanitize=address` | AddressSanitizer |
| `-fsanitize=undefined` | UndefinedBehaviorSanitizer |
| `-fstack-protector-strong` | Stack smashing protection |
| `-pie -fPIE` | Position-independent executable (ASLR) |
| `-D_FORTIFY_SOURCE=2` | Buffer overflow detection in glibc |
| `-I/path` | Add include search path |
| `-L/path` | Add library search path |
| `-lname` | Link against `libname` |

---

## Key Takeaways

- `#include` is copy-paste. Headers declare interfaces; source files define implementations.
- Always use include guards (`#ifndef` or `#pragma once`) to prevent double inclusion.
- Macro arguments are textually substituted — always parenthesise them. Prefer `static inline` for type safety.
- Each `.c` file compiles independently. The linker resolves cross-file references.
- Static libraries are self-contained; dynamic libraries are shared. Choose based on deployment needs.
- Compile with all warnings, sanitisers, and stack protection enabled during development.
