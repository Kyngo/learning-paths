---
title: "Pointers & Memory"
weight: 2
---

# Pointers & Memory

Pointers are the most powerful and most dangerous feature of C. A pointer is a variable that holds a memory address. Understanding pointers is understanding how the computer actually manages data.

---

## What Is a Pointer?

```c
int x = 42;
int *p = &x;   // p holds the address of x

printf("%d\n", x);    // 42 (the value)
printf("%p\n", (void*)p);  // 0x7ffd5e8c3a4c (the address)
printf("%d\n", *p);   // 42 (dereference — read value at address)

*p = 100;              // write through the pointer
printf("%d\n", x);    // 100 (x was modified via p)
```

| Operation | Syntax | Meaning |
|-----------|--------|---------|
| Declare pointer | `int *p` | p is a pointer to int |
| Address-of | `&x` | Get the address of x |
| Dereference | `*p` | Access the value at address p |
| NULL pointer | `NULL` or `(void*)0` | Points to nothing |

---

## Stack vs Heap

```
┌─────────────────────┐ High address
│       Stack         │ ← function locals, grows downward
│  (automatic, LIFO)  │
├─────────────────────┤
│         ↓           │
│    (free space)     │
│         ↑           │
├─────────────────────┤
│       Heap          │ ← malloc'd memory, grows upward
│  (dynamic, manual)  │
├─────────────────────┤
│   Data (BSS + init) │ ← globals, statics
├─────────────────────┤
│       Text          │ ← program code (read-only)
└─────────────────────┘ Low address
```

| Memory Region | Allocated By | Freed By | Lifetime |
|--------------|-------------|----------|----------|
| Stack | Function call | Function return | Automatic |
| Heap | `malloc()`, `calloc()` | `free()` | Manual |
| Data (global) | Program start | Program end | Program lifetime |
| Text (code) | Program load | Program end | Program lifetime |

---

## Dynamic Memory

```c
#include <stdlib.h>

// Allocate
int *arr = malloc(10 * sizeof(int));   // 10 ints, UNINITIALIZED
int *arr = calloc(10, sizeof(int));    // 10 ints, ZEROED

// Check for failure
if (arr == NULL) {
    perror("malloc failed");
    exit(1);
}

// Use
arr[0] = 42;
arr[9] = 100;

// Resize
arr = realloc(arr, 20 * sizeof(int)); // grow to 20 ints

// Free
free(arr);
arr = NULL;  // prevent use-after-free
```

### The Four Memory Bugs

| Bug | Cause | Consequence |
|-----|-------|-------------|
| Memory leak | `malloc` without `free` | Unbounded memory growth |
| Use-after-free | Dereference after `free` | Undefined behaviour (crash, corruption) |
| Double free | `free` the same pointer twice | Heap corruption |
| Buffer overflow | Write past allocated bounds | Stack smashing, code execution |

```c
// LEAK — forgot to free
void leak(void) {
    int *p = malloc(100);
    return;  // p is lost, 100 bytes leaked
}

// USE-AFTER-FREE
int *p = malloc(sizeof(int));
*p = 42;
free(p);
printf("%d\n", *p);  // UNDEFINED BEHAVIOUR

// DOUBLE FREE
free(p);
free(p);  // UNDEFINED BEHAVIOUR

// BUFFER OVERFLOW
int arr[5];
arr[10] = 42;  // writes beyond array bounds — UNDEFINED BEHAVIOUR
```

---

## Pointer Arithmetic

Pointers can be incremented and decremented. The step size is the size of the pointed-to type:

```c
int arr[] = {10, 20, 30, 40, 50};
int *p = arr;    // points to arr[0]

printf("%d\n", *p);       // 10
printf("%d\n", *(p + 1)); // 20 (next int = +4 bytes)
printf("%d\n", *(p + 2)); // 30

p++;
printf("%d\n", *p);       // 20

// arr[i] is syntactic sugar for *(arr + i)
```

### Pointer Subtraction

```c
int *start = &arr[0];
int *end = &arr[4];
ptrdiff_t diff = end - start;  // 4 (elements, not bytes)
```

---

## Arrays and Pointers

Arrays and pointers are closely related but not identical:

```c
int arr[5] = {1, 2, 3, 4, 5};
int *p = arr;    // array decays to pointer to first element

// These are equivalent:
arr[2]      // 3
*(arr + 2)  // 3
p[2]        // 3
*(p + 2)    // 3

// But:
sizeof(arr)  // 20 (5 × 4 bytes — size of the array)
sizeof(p)    // 8 (size of a pointer on 64-bit)
```

**Array decay:** When you pass an array to a function, it decays to a pointer — the function receives a pointer, not a copy of the array. It cannot know the array's length.

```c
void print_array(int *arr, size_t len) {
    for (size_t i = 0; i < len; i++) {
        printf("%d ", arr[i]);
    }
}

// ALWAYS pass the length separately
int data[] = {1, 2, 3};
print_array(data, 3);
```

---

## Pointers to Pointers

```c
int x = 42;
int *p = &x;      // pointer to int
int **pp = &p;     // pointer to pointer to int

printf("%d\n", **pp);  // 42

// Used for:
// 1. Modifying a pointer in a function
void allocate(int **out) {
    *out = malloc(sizeof(int));
    **out = 42;
}
int *result;
allocate(&result);

// 2. Arrays of strings (char **)
char *names[] = {"Alice", "Bob", "Carol"};
// names is char**
```

---

## Function Pointers

```c
// Declaration
int (*operation)(int, int);

// Assignment
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

operation = add;
printf("%d\n", operation(3, 4));  // 7

operation = sub;
printf("%d\n", operation(3, 4));  // -1

// As function parameter (callbacks)
void apply(int *arr, size_t len, int (*fn)(int)) {
    for (size_t i = 0; i < len; i++) {
        arr[i] = fn(arr[i]);
    }
}

int double_it(int x) { return x * 2; }
apply(data, 5, double_it);

// typedef for readability
typedef int (*BinaryOp)(int, int);
BinaryOp op = add;
```

---

## Void Pointers

`void*` is a generic pointer — it can point to any type but cannot be dereferenced directly:

```c
void *generic = malloc(sizeof(int));
int *specific = (int *)generic;  // cast to use
*specific = 42;

// Used in generic APIs:
void qsort(void *base, size_t nmemb, size_t size,
            int (*compar)(const void *, const void *));

int compare_ints(const void *a, const void *b) {
    return (*(const int *)a) - (*(const int *)b);
}
qsort(arr, len, sizeof(int), compare_ints);
```

---

## Common Idioms

### Null Check After Allocation

```c
int *data = malloc(n * sizeof(int));
if (!data) {
    fprintf(stderr, "out of memory\n");
    exit(EXIT_FAILURE);
}
```

### Free and Nullify

```c
free(ptr);
ptr = NULL;  // prevents use-after-free (subsequent deref is a clean crash)
```

### Sizeof with Variable, Not Type

```c
// GOOD — adapts if type changes
int *arr = malloc(n * sizeof(*arr));

// FRAGILE — type mismatch if arr changes to long*
int *arr = malloc(n * sizeof(int));
```

---

## Key Takeaways

- A pointer is a variable holding a memory address. `&` gets the address, `*` dereferences it.
- Stack memory is automatic (freed on function return). Heap memory is manual (`malloc`/`free`).
- Every `malloc` needs a `free`. Every `free` should nullify the pointer. Every allocation should be null-checked.
- Pointer arithmetic steps by the size of the pointed-to type, not by bytes.
- Arrays decay to pointers when passed to functions — always pass the length separately.
- Function pointers enable callbacks. `void*` enables generic programming (at the cost of type safety).
- Buffer overflows, use-after-free, and double-free are the three most common C security vulnerabilities.
