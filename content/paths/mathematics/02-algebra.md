---
title: "Algebra"
weight: 2
---

# Algebra

Algebra is the machinery for manipulating symbols and solving equations — the language in which algorithms express their relationships, complexity bounds are derived, and configuration parameters are computed.

---

## Expressions, Equations, and Identities

### Expressions

An **expression** is a mathematical phrase that combines numbers, variables, and operations. It does not have an equals sign.

```
3x + 7
n² - 1
log₂(n) + 1
```

### Equations

An **equation** asserts that two expressions are equal. It can be solved for unknowns.

```
3x + 7 = 22    →  x = 5
n² - 1 = 0     →  n = ±1
```

### Identities

An **identity** is an equation that holds for all values of the variables.

```
(a + b)² = a² + 2ab + b²           (always true)
a² - b² = (a + b)(a - b)           (always true)
```

---

## Manipulating Equations

Every equation manipulation must preserve equality — whatever you do to one side, do to the other.

| Operation | Example |
|-----------|---------|
| Add/subtract same value | `x - 3 = 7` → `x = 10` |
| Multiply/divide by same value | `2x = 14` → `x = 7` |
| Apply same function to both sides | `log(x) = 3` → `x = 10³ = 1000` |
| Square both sides (careful!) | `√x = 5` → `x = 25` |

**Warning:** Squaring both sides can introduce extraneous solutions. Always verify results.

### Systems of Equations

When you have multiple unknowns, you need as many independent equations as unknowns.

**Substitution method:**

```
Given:  x + y = 10
        2x - y = 5

From equation 1:  y = 10 - x
Substitute into 2: 2x - (10 - x) = 5
                    3x - 10 = 5
                    3x = 15
                    x = 5, y = 5
```

**Engineering application:** Solving for unknowns in capacity planning — if you know total requests and the ratio between read and write operations, you can solve for each.

---

## Exponents and Roots

### Exponent Rules

| Rule | Formula | Example |
|------|---------|---------|
| Product | aᵐ × aⁿ = aᵐ⁺ⁿ | 2³ × 2⁴ = 2⁷ = 128 |
| Quotient | aᵐ / aⁿ = aᵐ⁻ⁿ | 2⁷ / 2³ = 2⁴ = 16 |
| Power of power | (aᵐ)ⁿ = aᵐˣⁿ | (2³)² = 2⁶ = 64 |
| Zero exponent | a⁰ = 1 (a ≠ 0) | 5⁰ = 1 |
| Negative exponent | a⁻ⁿ = 1/aⁿ | 2⁻³ = 1/8 |
| Fractional exponent | a^(1/n) = ⁿ√a | 8^(1/3) = ∛8 = 2 |

### Why Exponents Matter in Computing

Exponential growth appears everywhere:

| Context | Growth | Example |
|---------|--------|---------|
| Binary tree levels | 2ⁿ nodes at depth n | Full binary tree with depth 20 has ~10⁶ leaves |
| Brute-force search | 2ⁿ combinations | 128-bit key: 2¹²⁸ ≈ 3.4 × 10³⁸ attempts |
| Bacterial growth model | 2ⁿ after n doublings | System load doubling every hour |
| Compound interest | (1 + r)ⁿ | Technical debt, resource consumption |

The "rule of 72": an investment growing at r% per period doubles in approximately 72/r periods. A service with 10% weekly traffic growth doubles in ~7 weeks.

---

## Logarithms

A logarithm answers: "to what power must I raise the base to get this number?"

```
log_b(x) = y  means  b^y = x
```

### Common Bases

| Notation | Base | Name | Use |
|----------|------|------|-----|
| log₂(x) | 2 | Binary logarithm | Algorithm complexity, binary search |
| ln(x) | e ≈ 2.718 | Natural logarithm | Calculus, ML, information theory |
| log₁₀(x) | 10 | Common logarithm | Orders of magnitude, decibels |

### Logarithm Rules

| Rule | Formula | Example |
|------|---------|---------|
| Product | log(ab) = log(a) + log(b) | log₂(32) = log₂(8) + log₂(4) = 3 + 2 = 5 |
| Quotient | log(a/b) = log(a) - log(b) | log₁₀(100/10) = 2 - 1 = 1 |
| Power | log(aⁿ) = n × log(a) | log₂(2¹⁰) = 10 × log₂(2) = 10 |
| Change of base | log_b(x) = log_c(x) / log_c(b) | log₂(100) = ln(100) / ln(2) ≈ 6.64 |
| Inverse | b^(log_b(x)) = x | 2^(log₂(8)) = 8 |

### Logarithms in Computer Science

| Context | Formula | Meaning |
|---------|---------|---------|
| Binary search | O(log₂ n) | Each step halves the search space |
| Balanced BST height | ⌈log₂(n + 1)⌉ | Minimum height for n nodes |
| Bits to represent n values | ⌈log₂ n⌉ | 256 values need 8 bits |
| Merge sort complexity | O(n log n) | Divide-and-conquer recursion depth |
| Information content | -log₂(p) bits | Rare events carry more information |
| pH scale | -log₁₀[H⁺] | Each unit is 10× concentration change |

### Key Intuition

Logarithms convert multiplication into addition and exponentiation into multiplication. This is why they appear naturally whenever:

- A problem is solved by **repeated halving** (binary search, divide-and-conquer)
- You need to count **how many digits** a number has (⌊log₁₀(n)⌋ + 1)
- You measure **orders of magnitude** (decibels, Richter scale, pH)

---

## Polynomials

A polynomial is an expression of the form:

```
p(x) = aₙxⁿ + aₙ₋₁xⁿ⁻¹ + ... + a₁x + a₀
```

The **degree** is the highest power of x with a nonzero coefficient.

### Common Polynomial Forms

| Degree | Name | Form | Example |
|--------|------|------|---------|
| 0 | Constant | a₀ | 7 |
| 1 | Linear | a₁x + a₀ | 3x + 2 |
| 2 | Quadratic | a₂x² + a₁x + a₀ | x² - 5x + 6 |
| 3 | Cubic | a₃x³ + ... + a₀ | x³ - 1 |

### The Quadratic Formula

For ax² + bx + c = 0:

```
x = (-b ± √(b² - 4ac)) / (2a)
```

The **discriminant** (b² - 4ac) determines the nature of solutions:

| Discriminant | Solutions |
|-------------|-----------|
| > 0 | Two distinct real roots |
| = 0 | One repeated real root |
| < 0 | Two complex conjugate roots |

### Factoring Common Patterns

```
a² - b²           = (a + b)(a - b)          Difference of squares
a² + 2ab + b²     = (a + b)²                Perfect square
a³ - b³           = (a - b)(a² + ab + b²)   Difference of cubes
x² + (a+b)x + ab  = (x + a)(x + b)          Factoring trinomials
```

**Engineering application:** Polynomial evaluation is at the heart of hash functions, CRC calculations, and interpolation. Horner's method evaluates a degree-n polynomial in O(n) operations:

```
p(x) = ((aₙx + aₙ₋₁)x + aₙ₋₂)x + ... + a₀
```

---

## Summation and Product Notation

Compact notation for expressing sums and products over a range.

### Sigma Notation (Summation)

```
 n
 Σ  f(i)  =  f(1) + f(2) + ... + f(n)
i=1
```

### Common Sums

| Sum | Closed Form | Example |
|-----|-------------|---------|
| Σᵢ₌₁ⁿ 1 | n | Sum of n ones |
| Σᵢ₌₁ⁿ i | n(n+1)/2 | 1 + 2 + ... + 100 = 5050 |
| Σᵢ₌₁ⁿ i² | n(n+1)(2n+1)/6 | Sum of squares |
| Σᵢ₌₀ⁿ rⁱ | (rⁿ⁺¹ - 1)/(r - 1) | Geometric series (r ≠ 1) |
| Σᵢ₌₀^∞ rⁱ | 1/(1 - r) for \|r\| < 1 | Infinite geometric series |
| Σᵢ₌₁ⁿ 1/i | ≈ ln(n) + γ | Harmonic series (γ ≈ 0.5772) |

### Why These Sums Matter

- **Σ i = n(n+1)/2** explains why nested loops give O(n²) complexity: iterating over all pairs of n elements.
- **Geometric series** explains why doubling-capacity arrays (ArrayList, Vec) have O(1) amortised insertion: 1 + 2 + 4 + ... + n = 2n - 1.
- **Harmonic series** appears in the analysis of quicksort's average case and the coupon collector problem.

### Pi Notation (Product)

```
 n
 Π  f(i)  =  f(1) × f(2) × ... × f(n)
i=1
```

**Factorial:** n! = Π_{i=1}^{n} i = 1 × 2 × ... × n

Factorials grow extremely fast:

| n | n! |
|---|-----|
| 5 | 120 |
| 10 | 3,628,800 |
| 20 | 2,432,902,008,176,640,000 |
| 52 | ~8 × 10⁶⁷ (deck of cards) |

---

## Inequalities

Inequalities express ordering relationships and are essential for algorithm analysis (Big-O bounds) and constraint satisfaction.

### Rules

| Rule | Constraint |
|------|-----------|
| Add/subtract same value from both sides | Always valid |
| Multiply/divide by positive value | Inequality direction preserved |
| Multiply/divide by negative value | **Inequality direction reverses** |
| Square both sides | Only valid when both sides are non-negative |

### The AM-GM Inequality

The arithmetic mean is always ≥ the geometric mean:

```
(a + b) / 2 ≥ √(ab)   for a, b ≥ 0
```

Equality holds when a = b. This is useful for optimisation: it tells you that for a fixed sum, the product is maximised when all terms are equal.

**Application:** Given a fixed memory budget split between cache and buffer, equal allocation maximises the product of their sizes — relevant when the performance benefit is proportional to the product.

### Triangle Inequality

```
|a + b| ≤ |a| + |b|
```

Generalised to metrics and distances, this is the fundamental property that makes algorithms like A* search correct.

---

## Absolute Value and Floor/Ceiling

### Absolute Value

```
|x| = x   if x ≥ 0
|x| = -x  if x < 0
```

Properties:
- |ab| = |a| × |b|
- |a + b| ≤ |a| + |b|  (triangle inequality)
- |a - b| represents the distance between a and b on the number line

### Floor and Ceiling

| Function | Symbol | Definition | Example |
|----------|--------|------------|---------|
| Floor | ⌊x⌋ | Largest integer ≤ x | ⌊3.7⌋ = 3, ⌊-2.3⌋ = -3 |
| Ceiling | ⌈x⌉ | Smallest integer ≥ x | ⌈3.2⌉ = 4, ⌈-2.7⌉ = -2 |

### Floor/Ceiling in Programming

```python
# Python
import math
math.floor(3.7)   # 3
math.ceil(3.2)    # 4
3.7 // 1          # 3.0 (floor division)

# Integer division IS floor division for positive numbers:
7 // 2            # 3 = ⌊7/2⌋

# But NOT for negative numbers in all languages:
# Python: -7 // 2 = -4 (floor)
# C/Java: -7 / 2  = -3 (truncation toward zero)
```

### Common Uses

| Pattern | Formula | Application |
|---------|---------|-------------|
| Pages needed for n items, k per page | ⌈n/k⌉ | Pagination |
| Levels in a complete binary tree | ⌊log₂(n)⌋ + 1 | Tree algorithms |
| Integer division rounding up | (n + k - 1) / k | Avoiding floating-point in C |
| Bits needed to represent n values | ⌈log₂(n)⌉ | Encoding |

---

## Key Takeaways

- Algebra provides the rules for manipulating equations — the same rules you use when deriving complexity bounds or solving configuration constraints.
- Exponents model explosive growth (brute-force search, resource consumption). Logarithms tame them by measuring "how many times do I halve?"
- Summation formulas (Σ i = n(n+1)/2, geometric series) directly explain algorithm complexity.
- Floor and ceiling functions appear in every pagination, tree height, or bit-width calculation.
- Master the logarithm rules — they are the single most useful algebraic tool in computer science.
