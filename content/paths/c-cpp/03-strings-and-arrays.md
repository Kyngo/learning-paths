---
title: "Strings & Arrays"
weight: 3
---

# Strings & Arrays

C has no string type. A "string" is a `char` array terminated by a null byte (`'\0'`). This simplicity is elegant but requires constant vigilance against buffer overflows.

---

## C Strings

```c
char greeting[] = "Hello";  // compiler adds '\0', size = 6
char *ptr = "World";        // points to string literal (read-only!)

// Memory layout of greeting:
// ['H','e','l','l','o','\0']
//  [0]  [1]  [2]  [3]  [4]  [5]
```

### String Length vs Array Size

```c
char s[] = "Hello";
strlen(s);    // 5 (characters, NOT including '\0')
sizeof(s);    // 6 (bytes, INCLUDING '\0')
```

### String Functions (`<string.h>`)

| Function | Purpose | Danger |
|----------|---------|--------|
| `strlen(s)` | Length (without '\0') | Scans until '\0' — O(n) |
| `strcpy(dst, src)` | Copy string | No bounds checking — use `strncpy` |
| `strncpy(dst, src, n)` | Copy at most n chars | May not null-terminate! |
| `strcat(dst, src)` | Concatenate | No bounds checking — use `strncat` |
| `strcmp(a, b)` | Compare (0 = equal) | Returns int, not bool |
| `strncmp(a, b, n)` | Compare first n chars | |
| `strchr(s, c)` | Find first occurrence of c | Returns NULL if not found |
| `strstr(s, sub)` | Find substring | Returns NULL if not found |
| `strtok(s, delim)` | Tokenise (destructive!) | Modifies the original string |
| `snprintf(buf, n, fmt, ...)` | Safe formatted print to buffer | **Preferred** — always bounds-checked |

### Safe String Handling

```c
// BAD — buffer overflow
char buf[10];
strcpy(buf, "This string is way too long for the buffer");

// GOOD — bounded
char buf[10];
snprintf(buf, sizeof(buf), "Hello %s", name);
// Always null-terminates, truncates if necessary
```

**Rule:** Always use `snprintf` instead of `sprintf`, `strncpy` instead of `strcpy`. Better yet, track buffer sizes explicitly.

---

## Multidimensional Arrays

```c
// 2D array (stack-allocated, contiguous)
int matrix[3][4] = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9, 10, 11, 12}
};

matrix[1][2]  // 7

// Array of pointers (each row can differ in length)
char *names[] = {"Alice", "Bob", "Carol"};
names[1]  // "Bob"
```

### Row-Major Order

C stores 2D arrays in row-major order (row by row in memory):

```c
int m[2][3] = {{1,2,3}, {4,5,6}};
// Memory: [1, 2, 3, 4, 5, 6]
//          row 0      row 1
```

Accessing `m[i][j]` is equivalent to `*(m + i * cols + j)`. Iterating row-first is cache-friendly; column-first causes cache misses.

---

## String Conversion

```c
#include <stdlib.h>

int n = atoi("42");           // string to int (no error checking!)
long l = strtol("42", NULL, 10);  // string to long (preferred — handles errors)

double d = atof("3.14");
double d = strtod("3.14", NULL);  // preferred

char buf[32];
snprintf(buf, sizeof(buf), "%d", 42);  // int to string
```

**Never use `atoi` in production** — it returns 0 on failure, which is indistinguishable from a valid input of "0". Use `strtol` with error checking.

---

## Buffer Overflow — The Classic C Vulnerability

```c
// The Heartbleed-class bug
char buf[64];
gets(buf);  // reads unlimited input — NEVER USE gets()

// The correct approach
char buf[64];
if (fgets(buf, sizeof(buf), stdin)) {
    buf[strcspn(buf, "\n")] = '\0';  // remove newline
}
```

`gets()` was removed from C11 because it is impossible to use safely. Always use `fgets()` with an explicit buffer size.

---

## Key Takeaways

- C strings are null-terminated `char` arrays. `strlen` is O(n) because it scans for `'\0'`.
- Always use bounded functions: `snprintf` not `sprintf`, `strncpy` not `strcpy`, `fgets` not `gets`.
- String literals (`"hello"`) are read-only. Writing to them is undefined behaviour.
- 2D arrays are row-major in memory. Row-first iteration is cache-friendly.
- Buffer overflows are C's most common security vulnerability. Always track and respect buffer sizes.
