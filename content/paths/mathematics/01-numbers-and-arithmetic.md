---
title: "Numbers & Arithmetic"
weight: 1
---

# Numbers & Arithmetic

Every computation a computer performs reduces to arithmetic on numbers. Understanding number systems — what they represent, how they are stored, and where they break — is the foundation of everything that follows.

---

## Number Systems

Mathematics organises numbers into nested sets, each extending the previous one to solve problems the smaller set cannot.

| Symbol | Name | Contains | Example |
|--------|------|----------|---------|
| ℕ | Natural numbers | 0, 1, 2, 3, … | Counting items in a list |
| ℤ | Integers | …, -2, -1, 0, 1, 2, … | Signed offsets, deltas |
| ℚ | Rational numbers | Any number expressible as p/q where p, q ∈ ℤ and q ≠ 0 | 1/3, 7/4, -22/7 |
| ℝ | Real numbers | All rationals plus irrationals (π, √2, e) | Continuous measurements |
| ℂ | Complex numbers | a + bi where a, b ∈ ℝ and i² = -1 | Signal processing, quantum computing |

### Why This Matters

- Array indices are natural numbers (ℕ).
- Database offsets and deltas are integers (ℤ).
- Financial calculations use rationals (ℚ) — never floating-point.
- Physics simulations operate on reals (ℝ), approximated by floats.
- Fourier transforms and quantum gates use complex numbers (ℂ).

### Key Properties

Every number system inherits the properties of the sets below it and adds new ones:

| Property | Example | Holds In |
|----------|---------|----------|
| Closure under addition | a + b is in the same set | ℕ, ℤ, ℚ, ℝ, ℂ |
| Closure under subtraction | a - b is in the same set | ℤ, ℚ, ℝ, ℂ (not ℕ: 3 - 5 = -2 ∉ ℕ) |
| Closure under division | a / b is in the same set | ℚ, ℝ, ℂ (not ℤ: 7 / 2 ∉ ℤ) |
| Additive identity | a + 0 = a | All |
| Multiplicative identity | a × 1 = a | All |
| Commutativity | a + b = b + a, a × b = b × a | All |
| Associativity | (a + b) + c = a + (b + c) | All |
| Distributivity | a × (b + c) = a × b + a × c | All |

---

## Positional Number Systems

Humans use base 10 out of biological accident (ten fingers). Computers use base 2 (voltage on/off). The same principles apply to any base.

### How Positional Notation Works

A number in base *b* with digits d₃d₂d₁d₀ represents:

```
d₃ × b³ + d₂ × b² + d₁ × b¹ + d₀ × b⁰
```

### Common Bases in Computing

| Base | Name | Digits | Prefix | Use Case |
|------|------|--------|--------|----------|
| 2 | Binary | 0, 1 | `0b` | Hardware, bitwise operations |
| 8 | Octal | 0–7 | `0o` | Unix file permissions |
| 10 | Decimal | 0–9 | (none) | Human-readable output |
| 16 | Hexadecimal | 0–9, A–F | `0x` | Memory addresses, colours, hashes |

### Conversion Examples

**Binary to decimal:**

```
0b1101 = 1×2³ + 1×2² + 0×2¹ + 1×2⁰
       = 8 + 4 + 0 + 1
       = 13
```

**Decimal to binary** (repeated division by 2):

```
13 ÷ 2 = 6 remainder 1  ↑
 6 ÷ 2 = 3 remainder 0  │  Read remainders
 3 ÷ 2 = 1 remainder 1  │  bottom to top
 1 ÷ 2 = 0 remainder 1  │
                         → 1101
```

**Hexadecimal to binary** (each hex digit = 4 bits):

```
0xFF = 1111 1111 = 255
0xCAFE = 1100 1010 1111 1110
```

### Powers of 2 Every Engineer Should Know

| Power | Value | Approximate | Common Name |
|-------|-------|-------------|-------------|
| 2⁰ | 1 | | |
| 2⁷ | 128 | | Signed byte max + 1 |
| 2⁸ | 256 | | Unsigned byte range |
| 2¹⁰ | 1,024 | ~10³ | 1 KiB |
| 2¹⁶ | 65,536 | ~65K | Port range |
| 2²⁰ | 1,048,576 | ~10⁶ | 1 MiB |
| 2³⁰ | 1,073,741,824 | ~10⁹ | 1 GiB |
| 2³² | 4,294,967,296 | ~4 × 10⁹ | IPv4 address space |
| 2⁶⁴ | ~1.8 × 10¹⁹ | | 64-bit address space |

---

## Modular Arithmetic

Modular arithmetic is arithmetic that "wraps around" after reaching a certain value, called the **modulus**. It is the mathematics of clocks, hash functions, and cryptography.

### Definition

Given integers a, b, and a positive integer n:

```
a ≡ b (mod n)  means  n divides (a - b)
```

Equivalently: a and b have the same remainder when divided by n.

### Examples

```
17 ≡ 2 (mod 5)     because 17 - 2 = 15, and 5 | 15
-3 ≡ 4 (mod 7)     because -3 - 4 = -7, and 7 | -7
100 ≡ 0 (mod 10)   because 100 - 0 = 100, and 10 | 100
```

### Properties

Modular arithmetic preserves addition, subtraction, and multiplication:

```
(a + b) mod n = ((a mod n) + (b mod n)) mod n
(a × b) mod n = ((a mod n) × (b mod n)) mod n
```

**Warning:** Division does not work directly in modular arithmetic. You need the **modular multiplicative inverse** (see cryptography applications).

### Engineering Applications

| Application | How Modular Arithmetic Is Used |
|-------------|-------------------------------|
| Hash tables | `index = hash(key) % table_size` |
| Circular buffers | `next = (current + 1) % capacity` |
| Load balancing | `server = request_id % num_servers` |
| Cryptography (RSA) | `ciphertext = message^e mod n` |
| Checksums | CRC, Luhn algorithm |
| Random number generators | Linear congruential: `x_{n+1} = (a × x_n + c) mod m` |
| Clock arithmetic | `(14 + 3) mod 12 = 5` (2 PM + 3 hours = 5 PM) |

### The Modulo Operator in Programming

Different languages handle negative numbers differently:

| Language | `-7 % 3` | Behaviour |
|----------|----------|-----------|
| Python | `2` | Result has the sign of the divisor (mathematical) |
| Java, C, JavaScript | `-1` | Result has the sign of the dividend (truncated) |
| Go | `-1` | Same as C |

To get a consistently positive result in languages with truncated division:

```
((a % n) + n) % n
```

---

## Integer Representation in Computers

Computers store integers in fixed-width binary. The encoding scheme determines the range of representable values.

### Unsigned Integers

All bits represent magnitude. An n-bit unsigned integer ranges from 0 to 2ⁿ - 1.

```
8-bit unsigned: 0 to 255
16-bit unsigned: 0 to 65,535
32-bit unsigned: 0 to 4,294,967,295
```

### Signed Integers (Two's Complement)

The most significant bit is the sign bit (0 = positive, 1 = negative). An n-bit signed integer ranges from -2ⁿ⁻¹ to 2ⁿ⁻¹ - 1.

```
8-bit signed: -128 to 127
32-bit signed: -2,147,483,648 to 2,147,483,647
64-bit signed: -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807
```

**How two's complement works:**

To negate a number: flip all bits and add 1.

```
 5 in 8-bit: 00000101
flip bits:    11111010
add 1:        11111011  → this is -5
```

**Why two's complement?** Addition works identically for signed and unsigned — the hardware does not need separate circuits.

### Overflow

When an arithmetic result exceeds the representable range, it **wraps around**:

```
127 + 1 = -128  (8-bit signed overflow)
255 + 1 = 0     (8-bit unsigned overflow)
```

This is a real source of bugs. The Ariane 5 rocket explosion (1996) was caused by a 64-bit float being converted to a 16-bit signed integer, causing an overflow that crashed the guidance system.

---

## Floating-Point Representation (IEEE 754)

Real numbers are stored as floating-point approximations. Understanding the representation prevents an entire class of subtle bugs.

### Structure

A 64-bit double consists of:

```
┌──────┬─────────────┬────────────────────────────────────────────────────┐
│ Sign │  Exponent   │                    Mantissa                       │
│ 1 bit│  11 bits    │                    52 bits                        │
└──────┴─────────────┴────────────────────────────────────────────────────┘

value = (-1)^sign × 2^(exponent - 1023) × 1.mantissa
```

### What Can Go Wrong

**Not all decimals are representable:**

```
0.1 + 0.2 = 0.30000000000000004  (in IEEE 754)
```

This is not a bug in your language — it is a fundamental limitation of binary fractions. 0.1 in binary is a repeating fraction (like 1/3 in decimal).

**Comparison traps:**

```python
# WRONG
if total == 0.3:
    ...

# RIGHT
if abs(total - 0.3) < 1e-9:
    ...
```

**Loss of significance:**

Subtracting nearly equal numbers amplifies relative error:

```
1.0000001 - 1.0000000 = 0.0000001
```

The result has only 1 significant digit despite both inputs having 8.

### Special Values

| Value | Meaning | Example |
|-------|---------|---------|
| `+0`, `-0` | Positive and negative zero | `1.0 / float('inf')` → `+0` |
| `Inf` | Positive infinity | `1.0 / 0.0` |
| `-Inf` | Negative infinity | `-1.0 / 0.0` |
| `NaN` | Not a Number | `0.0 / 0.0`, `sqrt(-1)` |

**NaN is the only value that is not equal to itself:** `NaN != NaN` is `true`. This is by specification.

### When Not to Use Floating-Point

| Use Case | Use Instead | Why |
|----------|-------------|-----|
| Money | Integer cents, `Decimal` type | Exact representation required |
| Timestamps | Integer nanoseconds since epoch | Precision loss over large ranges |
| Equality checks | Epsilon comparison or exact rationals | Representation error |
| Cryptographic operations | Arbitrary-precision integers | Determinism required |

---

## Bitwise Operations

Operations that work directly on the binary representation of integers. Fast, low-level, and used extensively in systems programming, networking, and cryptography.

| Operation | Symbol | Example (binary) | Result |
|-----------|--------|-------------------|--------|
| AND | `&` | `1010 & 1100` | `1000` |
| OR | `\|` | `1010 \| 1100` | `1110` |
| XOR | `^` | `1010 ^ 1100` | `0110` |
| NOT | `~` | `~1010` | `0101` |
| Left shift | `<<` | `0011 << 2` | `1100` |
| Right shift | `>>` | `1100 >> 2` | `0011` |

### Common Patterns

```python
# Check if n is even
n & 1 == 0

# Check if n is a power of 2
n > 0 and (n & (n - 1)) == 0

# Multiply by 2^k
n << k

# Divide by 2^k (integer division)
n >> k

# Set bit i
n | (1 << i)

# Clear bit i
n & ~(1 << i)

# Toggle bit i
n ^ (1 << i)

# Swap without temporary variable
a ^= b; b ^= a; a ^= b

# Extract lowest set bit
n & (-n)
```

### Subnet Masks as Bitwise AND

A subnet mask determines which bits of an IP address identify the network:

```
IP:     192.168.1.42   = 11000000.10101000.00000001.00101010
Mask:   255.255.255.0  = 11111111.11111111.11111111.00000000
AND:    192.168.1.0    = 11000000.10101000.00000001.00000000  (network)
```

---

## Key Takeaways

- Number systems form a hierarchy (ℕ ⊂ ℤ ⊂ ℚ ⊂ ℝ ⊂ ℂ), each solving limitations of the previous.
- Every positional number system uses the same principle: digit × base^position.
- Modular arithmetic underpins hash tables, circular buffers, load balancing, and cryptography.
- Two's complement is the universal signed integer encoding — overflow wraps silently.
- IEEE 754 floating-point cannot represent all decimals exactly. Never use floats for money or equality checks.
- Bitwise operations are the fundamental building blocks of systems programming and networking.
