---
title: "C Fundamentals"
weight: 1
---

# C Fundamentals

C is a small language — about 30 keywords, no classes, no exceptions, no garbage collector. It compiles to machine code, gives you direct hardware access, and trusts you to manage everything yourself. This is both its power and its danger.

---

## Hello World

```c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

```bash
gcc -o hello hello.c
./hello
```

### Anatomy

| Part | Purpose |
|------|---------|
| `#include <stdio.h>` | Preprocessor directive — includes standard I/O declarations |
| `int main(void)` | Entry point, returns int (0 = success, nonzero = error) |
| `printf()` | Formatted output to stdout |
| `return 0` | Exit code to the OS |

---

## Compilation Pipeline

```
source.c → [Preprocessor] → source.i → [Compiler] → source.s → [Assembler] → source.o → [Linker] → executable
```

| Stage | Tool | What It Does |
|-------|------|-------------|
| Preprocessing | `cpp` | Expands `#include`, `#define`, conditional compilation |
| Compilation | `cc1` | Translates C to assembly |
| Assembly | `as` | Translates assembly to machine code (object file) |
| Linking | `ld` | Combines object files + libraries into executable |

```bash
gcc -E hello.c > hello.i    # preprocess only
gcc -S hello.c               # compile to assembly
gcc -c hello.c               # compile to object file
gcc -o hello hello.c         # full pipeline
```

### Compiler Flags You Should Always Use

```bash
gcc -Wall -Wextra -Werror -std=c17 -pedantic -O2 -o myapp main.c
```

| Flag | Purpose |
|------|---------|
| `-Wall` | Enable most warnings |
| `-Wextra` | Additional warnings |
| `-Werror` | Treat warnings as errors |
| `-std=c17` | Use C17 standard |
| `-pedantic` | Strict standard compliance |
| `-O2` | Optimisation level 2 |
| `-g` | Include debug symbols |
| `-fsanitize=address` | AddressSanitizer (catches memory bugs) |

---

## Types

### Integer Types

| Type | Minimum Size | Typical Size | Range (typical) |
|------|-------------|-------------|----------------|
| `char` | 1 byte | 1 byte | -128 to 127 or 0 to 255 |
| `short` | 2 bytes | 2 bytes | -32,768 to 32,767 |
| `int` | 2 bytes | 4 bytes | -2³¹ to 2³¹-1 |
| `long` | 4 bytes | 4 or 8 bytes | Platform-dependent |
| `long long` | 8 bytes | 8 bytes | -2⁶³ to 2⁶³-1 |

**Use fixed-width types** from `<stdint.h>` for portability:

```c
#include <stdint.h>

int8_t   a;   // exactly 8 bits, signed
uint32_t b;   // exactly 32 bits, unsigned
int64_t  c;   // exactly 64 bits, signed
size_t   len; // unsigned, big enough for any object size
```

### Floating-Point Types

| Type | Size | Precision |
|------|------|-----------|
| `float` | 4 bytes | ~7 decimal digits |
| `double` | 8 bytes | ~15 decimal digits |
| `long double` | 8-16 bytes | Platform-dependent |

### Boolean

```c
#include <stdbool.h>

bool is_valid = true;
bool is_empty = false;
```

Before C99: use `int` with 0/nonzero convention.

---

## Variables and Scope

```c
int global = 42;              // file scope (avoid)

void function(void) {
    int local = 10;           // block scope
    static int persistent = 0; // persists across calls
    persistent++;

    {
        int inner = 5;        // nested block scope
    }
    // inner is not accessible here
}
```

### Storage Classes

| Specifier | Scope | Lifetime | Use |
|-----------|-------|----------|-----|
| `auto` (default) | Block | Block | Local variables |
| `static` (local) | Block | Program | Persistent local state |
| `static` (global) | File | Program | File-private globals |
| `extern` | Global | Program | Declare variable defined elsewhere |
| `register` | Block | Block | Hint: keep in CPU register (obsolete) |

---

## Printf Format Specifiers

```c
printf("%d\n", 42);           // int
printf("%u\n", 42u);          // unsigned int
printf("%ld\n", 42L);         // long
printf("%lld\n", 42LL);       // long long
printf("%zu\n", sizeof(int)); // size_t
printf("%f\n", 3.14);         // double
printf("%.2f\n", 3.14159);   // 2 decimal places
printf("%e\n", 3.14e10);     // scientific notation
printf("%c\n", 'A');          // char
printf("%s\n", "hello");     // string
printf("%p\n", (void*)ptr);  // pointer address
printf("%x\n", 255);         // hex (ff)
printf("%o\n", 255);         // octal (377)
printf("%%\n");              // literal %
```

**Warning:** Mismatched format specifiers are undefined behaviour. Use `-Wformat` (included in `-Wall`) to catch them.

---

## Control Flow

### If/Else

```c
if (x > 0) {
    printf("positive\n");
} else if (x == 0) {
    printf("zero\n");
} else {
    printf("negative\n");
}
```

### For Loop

```c
for (int i = 0; i < 10; i++) {
    printf("%d\n", i);
}
```

### While / Do-While

```c
while (condition) {
    // body
}

do {
    // body (executes at least once)
} while (condition);
```

### Switch

```c
switch (option) {
    case 1:
        printf("one\n");
        break;          // REQUIRED — fallthrough is default
    case 2:
    case 3:
        printf("two or three\n");
        break;
    default:
        printf("other\n");
}
```

---

## Functions

```c
// Declaration (prototype)
int add(int a, int b);

// Definition
int add(int a, int b) {
    return a + b;
}

// void = no return value
void greet(const char* name) {
    printf("Hello, %s!\n", name);
}
```

### Pass by Value

C passes everything by value — including pointers (the pointer is copied, not the pointed-to data):

```c
void increment(int x) {
    x++;  // modifies the local copy only
}

int n = 5;
increment(n);
printf("%d\n", n);  // still 5

// To modify the original: pass a pointer
void increment_ptr(int *x) {
    (*x)++;
}
increment_ptr(&n);
printf("%d\n", n);  // now 6
```

---

## The `const` Qualifier

```c
const int MAX = 100;        // value cannot be changed
const char* name = "Alice"; // pointer to constant data (data is immutable)
char* const ptr = buffer;   // constant pointer (pointer itself is immutable)
const char* const both = "fixed"; // both immutable
```

**Read declarations right to left:** `const char*` = "pointer to const char" (the char is const). `char* const` = "const pointer to char" (the pointer is const).

---

## Key Takeaways

- C is small and explicit. There is no hidden magic — what you write is what runs.
- Always compile with `-Wall -Wextra -Werror`. Warnings in C are often real bugs.
- Use `<stdint.h>` for fixed-width integer types. Never rely on `int` being a specific size.
- Everything is pass-by-value. To modify an argument, pass a pointer to it.
- Printf format mismatches are undefined behaviour. The compiler warns about them if you let it.
- `const` correctness communicates intent and catches accidental modification.
