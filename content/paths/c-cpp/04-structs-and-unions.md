---
title: "Structs & Unions"
weight: 4
---

# Structs & Unions

Structs group related data into a single type. Unions overlay multiple fields in the same memory. Together with enums and bitfields, they let you model any data layout — from database records to hardware registers.

---

## Structs

```c
struct Point {
    double x;
    double y;
};

struct Point origin = {0.0, 0.0};         // positional init
struct Point p = {.x = 3.0, .y = 4.0};   // designated init (C99)

printf("(%f, %f)\n", p.x, p.y);

// Pointer access
struct Point *pp = &p;
printf("(%f, %f)\n", pp->x, pp->y);   // -> dereferences + accesses
```

### Typedef

```c
typedef struct {
    int id;
    char name[64];
    double salary;
} Employee;

Employee e = {1, "Alice", 95000.0};
```

### Structs with Pointers

```c
typedef struct Node {
    int value;
    struct Node *next;  // self-referential — must use full struct name
} Node;

Node a = {1, NULL};
Node b = {2, &a};
// b -> a -> NULL
```

---

## Memory Layout and Alignment

The compiler inserts **padding** to align struct fields to their natural boundaries:

```c
struct Padded {
    char a;     // 1 byte
    // 3 bytes padding (to align int to 4-byte boundary)
    int b;      // 4 bytes
    char c;     // 1 byte
    // 3 bytes padding (struct size rounds up to multiple of largest alignment)
};
// sizeof(struct Padded) = 12 (not 6!)

struct Packed {
    int b;      // 4 bytes
    char a;     // 1 byte
    char c;     // 1 byte
    // 2 bytes padding
};
// sizeof(struct Packed) = 8 — better layout, same data
```

**Rule:** Order fields from largest to smallest to minimise padding.

### Checking Layout

```c
#include <stddef.h>

printf("size: %zu\n", sizeof(struct Padded));
printf("offset of b: %zu\n", offsetof(struct Padded, b));
```

### Packed Structs (Disable Padding)

```c
struct __attribute__((packed)) Wire {
    uint8_t type;
    uint32_t length;
    uint8_t data[];
};
// sizeof = 5 (no padding — but unaligned access may be slower)
```

Use packed structs for network protocols and file formats where the byte layout must be exact.

---

## Unions

A union stores all fields at the **same** memory location. Only one field is valid at a time:

```c
union Value {
    int i;
    double d;
    char s[16];
};

union Value v;
v.i = 42;
printf("%d\n", v.i);    // 42
v.d = 3.14;
printf("%d\n", v.i);    // garbage — i is no longer valid
printf("%f\n", v.d);    // 3.14

sizeof(union Value);     // 16 (size of the largest member)
```

### Tagged Unions (Discriminated Unions)

Combine an enum with a union to track which field is active:

```c
typedef enum { TYPE_INT, TYPE_DOUBLE, TYPE_STRING } ValueType;

typedef struct {
    ValueType type;
    union {
        int i;
        double d;
        char s[32];
    } data;
} Value;

Value v = {.type = TYPE_INT, .data.i = 42};

switch (v.type) {
    case TYPE_INT:    printf("%d\n", v.data.i); break;
    case TYPE_DOUBLE: printf("%f\n", v.data.d); break;
    case TYPE_STRING: printf("%s\n", v.data.s); break;
}
```

This is the C equivalent of Rust's `enum` with data or TypeScript's discriminated unions.

---

## Enums

```c
enum Color { RED, GREEN, BLUE };              // 0, 1, 2
enum Status { OK = 200, NOT_FOUND = 404 };    // explicit values

enum Color c = GREEN;
printf("%d\n", c);  // 1
```

C enums are just integers — no type safety. You can assign any int to an enum variable.

---

## Bitfields

Pack multiple small values into a single integer — common in hardware registers and protocol headers:

```c
struct Flags {
    unsigned int readable  : 1;  // 1 bit
    unsigned int writable  : 1;  // 1 bit
    unsigned int executable: 1;  // 1 bit
    unsigned int reserved  : 5;  // 5 bits (padding)
};

struct Flags f = {.readable = 1, .writable = 1, .executable = 0};
sizeof(struct Flags);  // 4 (minimum allocation unit)
```

### Bitfields vs Manual Bit Manipulation

```c
// Bitfield approach (readable)
struct Flags f;
f.readable = 1;

// Manual approach (portable, predictable layout)
uint8_t flags = 0;
flags |= (1 << 0);  // set readable
flags &= ~(1 << 1); // clear writable
if (flags & (1 << 2)) { /* executable */ }
```

Bitfield layout is implementation-defined (bit ordering, padding). For network protocols and file formats, use manual bit manipulation for portability.

---

## Flexible Array Members (C99)

A struct with a variable-length trailing array:

```c
typedef struct {
    size_t length;
    char data[];  // flexible array member — must be last
} Buffer;

Buffer *buf = malloc(sizeof(Buffer) + 100);
buf->length = 100;
memcpy(buf->data, input, 100);

free(buf);
```

Used in network packets, database pages, and any structure with variable-length payload.

---

## Key Takeaways

- Structs group related data. Order fields largest-to-smallest to minimise alignment padding.
- Unions overlay fields in the same memory — only one is valid at a time. Use tagged unions for safety.
- C enums are integers with no type safety. Use them for named constants.
- Bitfields pack flags into minimal space but have implementation-defined layout. Use manual bit ops for portable binary formats.
- `sizeof` tells you the true size (with padding). `offsetof` tells you where each field starts.
- Flexible array members (`char data[]`) enable variable-length structs without extra indirection.
