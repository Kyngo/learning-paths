---
title: "Functions & Graphs"
weight: 3
---

# Functions & Graphs

A function is a rule that assigns exactly one output to each input. Functions are the abstraction behind every algorithm's behaviour, every complexity analysis, and every mathematical model in engineering.

---

## What Is a Function?

Formally, a function f: A → B is a mapping from a set A (the **domain**) to a set B (the **codomain**) where every element of A maps to exactly one element of B.

```
f: ℝ → ℝ
f(x) = x²

Domain: all real numbers (inputs)
Codomain: all real numbers (declared output type)
Range: [0, ∞) (actual outputs — a subset of the codomain)
```

### Function vs. Relation

A **relation** can map one input to multiple outputs. A **function** cannot.

```
y = x²     → function (each x gives one y)
x² + y² = 1  → relation (each x in (-1,1) gives two y values)
```

### The Vertical Line Test

On a graph, a curve represents a function if and only if every vertical line intersects it at most once.

---

## Function Composition

Applying one function after another:

```
(g ∘ f)(x) = g(f(x))
```

Read "g of f of x" — f is applied first, then g.

```
f(x) = 2x + 1
g(x) = x²

(g ∘ f)(x) = g(f(x)) = g(2x + 1) = (2x + 1)²
(f ∘ g)(x) = f(g(x)) = f(x²) = 2x² + 1
```

**Note:** Composition is generally not commutative: g ∘ f ≠ f ∘ g.

### In Programming

Function composition is pipe/chain:

```python
# Python
result = g(f(x))

# Bash
cat file | sort | uniq     # uniq(sort(cat(file)))

# Functional style
compose = lambda f, g: lambda x: g(f(x))
```

---

## Inverse Functions

If f maps x to y, then f⁻¹ maps y back to x:

```
f(x) = 2x + 3
f⁻¹(x) = (x - 3) / 2

f(f⁻¹(x)) = x  and  f⁻¹(f(x)) = x
```

A function has an inverse if and only if it is **bijective** (one-to-one and onto).

### Common Inverses

| Function | Inverse | Domain Restriction |
|----------|---------|-------------------|
| f(x) = x² | f⁻¹(x) = √x | x ≥ 0 |
| f(x) = eˣ | f⁻¹(x) = ln(x) | x > 0 |
| f(x) = 2ˣ | f⁻¹(x) = log₂(x) | x > 0 |
| f(x) = 10ˣ | f⁻¹(x) = log₁₀(x) | x > 0 |
| f(x) = sin(x) | f⁻¹(x) = arcsin(x) | -1 ≤ x ≤ 1 |

**Engineering relevance:** Encryption and decryption are inverse functions. Encoding and decoding. Compression and decompression. Serialisation and deserialisation.

---

## Piecewise Functions

A function defined by different expressions on different intervals.

```
       ⎧ 0        if x < 0
f(x) = ⎨ x        if 0 ≤ x ≤ 1
       ⎩ 1        if x > 1
```

### In Programming

```python
def clamp(x, lo=0, hi=1):
    if x < lo: return lo
    if x > hi: return hi
    return x
```

### ReLU (Rectified Linear Unit)

The most popular activation function in neural networks is piecewise:

```
ReLU(x) = max(0, x) = ⎧ 0   if x ≤ 0
                       ⎩ x   if x > 0
```

---

## Floor and Ceiling Functions

Covered in detail in the Algebra section. As functions:

```
⌊x⌋: ℝ → ℤ  (floor — round down)
⌈x⌉: ℝ → ℤ  (ceiling — round up)
```

Both are **step functions** — piecewise constant, with jumps at integer values.

The **fractional part** is defined as:

```
{x} = x - ⌊x⌋
```

For example: {3.7} = 0.7, {-2.3} = 0.7 (not -0.3).

---

## Growth Rates

Understanding how functions grow as their input increases is the core skill for algorithm analysis.

### Common Growth Rates (Slowest to Fastest)

| Growth | Name | Example | Intuition |
|--------|------|---------|-----------|
| O(1) | Constant | Hash lookup | Doesn't grow at all |
| O(log n) | Logarithmic | Binary search | Grows by 1 when n doubles |
| O(√n) | Square root | Trial division | Grows slowly, faster than log |
| O(n) | Linear | Single scan | Doubles when n doubles |
| O(n log n) | Linearithmic | Merge sort | Slightly superlinear |
| O(n²) | Quadratic | Nested loops | 4× when n doubles |
| O(n³) | Cubic | Matrix multiplication (naive) | 8× when n doubles |
| O(2ⁿ) | Exponential | Subsets, brute force | Doubles with each +1 to n |
| O(n!) | Factorial | Permutations | Grows faster than exponential |

### Concrete Numbers

How long each complexity takes for different input sizes (assuming 10⁹ operations/second):

| n | log₂ n | n | n log₂ n | n² | 2ⁿ |
|---|--------|---|----------|-----|-----|
| 10 | 3 | 10 | 33 | 100 | 1,024 |
| 100 | 7 | 100 | 664 | 10,000 | 1.3 × 10³⁰ |
| 1,000 | 10 | 1,000 | 9,966 | 10⁶ | ∞ |
| 10⁶ | 20 | 10⁶ | 2 × 10⁷ | 10¹² | ∞ |
| 10⁹ | 30 | 10⁹ | 3 × 10¹⁰ | 10¹⁸ | ∞ |

At n = 10⁶:
- O(n log n) takes ~0.02 seconds
- O(n²) takes ~16 minutes
- O(2ⁿ) would take longer than the age of the universe

### Dominance Rules

When summing terms, only the fastest-growing term matters for large n:

```
n³ + 100n² + 5000n → O(n³)
2ⁿ + n¹⁰⁰ → O(2ⁿ)
3n log n + 7n → O(n log n)
```

Constant factors are dropped because they don't affect the growth rate:

```
5n² → O(n²)
```

---

## Important Named Functions

### Linear: f(x) = mx + b

- m is the slope (rate of change)
- b is the y-intercept
- Graph is a straight line
- O(n) algorithms have linear growth

### Quadratic: f(x) = ax² + bx + c

- Graph is a parabola (opens up if a > 0, down if a < 0)
- Vertex at x = -b/(2a)
- O(n²) algorithms have quadratic growth

### Exponential: f(x) = aˣ (a > 1)

- Grows faster than any polynomial
- Key property: f(x + 1) / f(x) = a (constant ratio)
- Doubling time: T_double = ln(2) / ln(a)
- Modelling: population growth, compound interest, viral spread

### Logarithmic: f(x) = log_b(x)

- Inverse of exponential
- Grows slower than any positive power of x
- Key property: log(2n) = log(n) + 1
- Modelling: perceived loudness, earthquake intensity, algorithm complexity

### Sigmoid: f(x) = 1 / (1 + e⁻ˣ)

- S-shaped curve, output bounded to (0, 1)
- Used in logistic regression, neural network activation
- At x = 0: f(0) = 0.5
- Approaches 0 as x → -∞, approaches 1 as x → +∞

### Softmax

Generalisation of sigmoid to multiple classes:

```
softmax(xᵢ) = eˣⁱ / Σⱼ eˣʲ
```

Converts a vector of real numbers into a probability distribution. Used in the final layer of classification networks and in attention mechanisms (transformers).

---

## Asymptotic Behaviour

How a function behaves as its input approaches infinity (or some critical point).

### Limits at Infinity

```
lim (x→∞) 1/x = 0              (approaches zero)
lim (x→∞) (1 + 1/x)ˣ = e      (definition of e)
lim (x→∞) log(x)/x = 0         (log grows slower than linear)
lim (x→∞) x/2ˣ = 0             (exponential dominates any polynomial)
```

### The Hierarchy of Growth

```
1 << log(log n) << log n << √n << n << n log n << n² << n³ << 2ⁿ << n! << nⁿ
```

This hierarchy is the backbone of Big-O analysis. If you remember nothing else from this section, remember this chain.

---

## Sequences and Series

A **sequence** is an ordered list of numbers following a rule. A **series** is the sum of a sequence's terms.

### Arithmetic Sequences

Each term differs from the previous by a constant d:

```
a, a+d, a+2d, ..., a+(n-1)d

Sum = n × (first + last) / 2 = n(2a + (n-1)d) / 2
```

Example: 1, 3, 5, 7, 9 → sum = 5 × (1+9)/2 = 25

### Geometric Sequences

Each term is a constant ratio r times the previous:

```
a, ar, ar², ..., arⁿ⁻¹

Sum = a × (1 - rⁿ) / (1 - r)    (r ≠ 1)
```

If |r| < 1, the infinite sum converges:

```
Sum = a / (1 - r)
```

Example: 1 + 1/2 + 1/4 + 1/8 + ... = 1/(1 - 1/2) = 2

### Fibonacci Sequence

```
F(0) = 0, F(1) = 1
F(n) = F(n-1) + F(n-2)

0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...
```

Growth rate: F(n) ≈ φⁿ/√5, where φ = (1 + √5)/2 ≈ 1.618 (the golden ratio).

The naive recursive implementation is O(2ⁿ); memoised is O(n); matrix exponentiation gives O(log n).

---

## Parametric and Implicit Forms

### Parametric

Coordinates expressed as functions of a parameter t:

```
x(t) = cos(t)
y(t) = sin(t)     →  traces a unit circle as t goes from 0 to 2π
```

Used in: animation paths, Bézier curves, trajectory modelling.

### Implicit

A relationship between x and y without solving for one in terms of the other:

```
x² + y² = r²      (circle of radius r)
```

Used in: collision detection, level sets, signed distance functions.

---

## Key Takeaways

- A function maps each input to exactly one output. The domain, codomain, and range are distinct concepts.
- Composition chains functions: g(f(x)). This is the mathematical model of pipelines.
- Growth rates form a strict hierarchy: 1 << log n << n << n log n << n² << 2ⁿ << n!
- Logarithms grow slower than any polynomial. Exponentials grow faster than any polynomial.
- Summation of arithmetic and geometric series explains the complexity of loops and amortised data structures.
- Named functions (sigmoid, softmax, ReLU) recur throughout ML — recognise their shapes and properties.
