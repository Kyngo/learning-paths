---
title: "Mathematics in Practice"
weight: 12
---

# Mathematics in Practice

This section bridges mathematical theory to engineering reality — the numerical traps that bite in production, the probabilistic methods that solve intractable problems, and the mathematical principles hiding inside everyday tools.

---

## Numerical Stability

### The Fundamental Problem

Computers work with finite-precision arithmetic. Every floating-point operation can introduce a small error, and these errors accumulate. Code that is mathematically correct can be numerically wrong.

### Catastrophic Cancellation

Subtracting nearly equal numbers amplifies relative error:

```python
# Mathematically: (1 + 10⁻¹⁵) - 1 = 10⁻¹⁵
a = 1.0 + 1e-15
b = 1.0
print(a - b)  # 1.1102230246251565e-15 (wrong!)
               # True value: 1e-15
```

The relative error is 11% — from a single subtraction.

### Summing Large Arrays

```python
# BAD: Naive summation of many small numbers
total = 0.0
for x in million_small_numbers:
    total += x  # Error accumulates with each addition

# BETTER: Kahan compensated summation
total = 0.0
compensation = 0.0
for x in million_small_numbers:
    y = x - compensation
    t = total + y
    compensation = (t - total) - y
    total = t

# BEST: Use the language's built-in
import math
total = math.fsum(million_small_numbers)  # Exact to full precision
```

### Log-Sum-Exp Trick

Computing softmax or log-probabilities requires summing exponentials, which easily overflow:

```python
# WRONG: Overflow for large values
import numpy as np
x = np.array([1000, 1001, 1002])
np.exp(x)  # [inf, inf, inf]

# RIGHT: Subtract the max first
x_max = np.max(x)
log_sum = x_max + np.log(np.sum(np.exp(x - x_max)))
# Mathematically identical, numerically stable
```

This trick is used inside every softmax implementation in every ML framework.

### Common Numerical Pitfalls

| Pitfall | Example | Fix |
|---------|---------|-----|
| Float equality | `0.1 + 0.2 != 0.3` | Use `abs(a - b) < epsilon` |
| Accumulation error | Summing 10⁶ floats | Kahan summation or `math.fsum` |
| Overflow in exp | `exp(1000) = inf` | Log-sum-exp trick |
| Underflow in probabilities | `0.001^1000 = 0` | Work in log-space |
| Loss of significance | `f(x+h) - f(x)` for small h | Rewrite algebraically |
| Integer overflow | `INT_MAX + 1` wraps | Use checked arithmetic or wider types |
| Division by zero | `1.0 / 0.0 = Inf` | Guard or use `nan`-propagation |

### The Golden Rule

**Work in log-space whenever possible.** Products become sums, powers become multiplications, and underflow/overflow become manageable.

```python
# Instead of: probability = p1 * p2 * p3 * ... * pn  (underflows to 0)
# Use:        log_prob = log(p1) + log(p2) + ... + log(pn)  (stays finite)
```

---

## Floating-Point Formats

### IEEE 754 Summary

| Format | Bits | Significand | Exponent | Range | Precision |
|--------|------|------------|----------|-------|-----------|
| half (fp16) | 16 | 10 bits | 5 bits | ±6.5 × 10⁴ | ~3 decimal digits |
| bfloat16 | 16 | 7 bits | 8 bits | ±3.4 × 10³⁸ | ~2 decimal digits |
| single (fp32) | 32 | 23 bits | 8 bits | ±3.4 × 10³⁸ | ~7 decimal digits |
| double (fp64) | 64 | 52 bits | 11 bits | ±1.8 × 10³⁰⁸ | ~15 decimal digits |

### Mixed Precision in ML

Modern ML training uses mixed precision: forward pass in fp16/bf16 (fast, fits in GPU memory), gradient accumulation in fp32 (maintains accuracy). bfloat16 is preferred over fp16 because its wider exponent range avoids overflow during training.

---

## Monte Carlo Methods

Monte Carlo methods use random sampling to solve problems that are deterministic but intractable by exhaustive computation.

### Estimating π

```python
import random

inside = 0
total = 1_000_000
for _ in range(total):
    x, y = random.random(), random.random()
    if x**2 + y**2 <= 1:
        inside += 1

pi_estimate = 4 * inside / total  # ≈ 3.14159...
```

Drop random points in a unit square. The fraction that land inside the quarter-circle times 4 gives π.

### Monte Carlo Integration

For high-dimensional integrals where grid methods are impractical:

```
∫ f(x) dx ≈ (1/N) Σ f(xᵢ)   where xᵢ are random samples
```

**Error:** O(1/√N) — independent of dimensionality. This is why Monte Carlo scales to high dimensions while grid methods scale as O(1/N^(1/d)), which is useless for d > 5.

### Applications

| Application | What Is Sampled |
|-------------|----------------|
| Rendering (path tracing) | Light paths through a scene |
| Financial modelling | Asset price trajectories |
| Reliability analysis | Component failure scenarios |
| Bayesian inference (MCMC) | Posterior distribution samples |
| Game AI (MCTS) | Game state trajectories |
| Risk analysis | Attack scenarios, failure modes |

### Variance Reduction

Naive Monte Carlo can be slow. Techniques to reduce the number of samples needed:

| Technique | Idea |
|-----------|------|
| Importance sampling | Sample more from important regions |
| Stratified sampling | Divide domain into strata, sample each |
| Antithetic variates | Use negatively correlated pairs |
| Control variates | Subtract a known quantity to reduce variance |

---

## Hashing Mathematics

### What Makes a Good Hash Function?

| Property | Mathematical Formulation |
|----------|------------------------|
| Uniform distribution | P(hash(x) = k) ≈ 1/m for any x, any bucket k |
| Avalanche effect | Changing 1 input bit changes ~50% of output bits |
| Collision resistance | Finding x ≠ y with hash(x) = hash(y) should be hard |
| Preimage resistance | Given h, finding x with hash(x) = h should be hard |

### Birthday Bound on Collisions

In a hash space of size N, expect the first collision after approximately √(πN/2) random insertions:

| Hash bits | Space N | Collision after ~√N | |
|-----------|---------|--------------------|-|
| 32 | 4.3 × 10⁹ | ~77,000 | Too small for large datasets |
| 64 | 1.8 × 10¹⁹ | ~5 × 10⁹ | Adequate for most uses |
| 128 | 3.4 × 10³⁸ | ~1.8 × 10¹⁹ | Secure against brute force |
| 256 | 1.2 × 10⁷⁷ | ~3.4 × 10³⁸ | Quantum-resistant |

### Universal Hashing

A family H of hash functions is **universal** if for any x ≠ y:

```
P_{h ∈ H}(h(x) = h(y)) ≤ 1/m
```

Choosing a random hash function from a universal family guarantees expected O(1) hash table operations regardless of the input distribution. This defeats adversarial inputs.

### Consistent Hashing

Maps both keys and nodes to a ring of size 2³². Each key is assigned to the nearest node clockwise:

- Adding a node: only keys between the new node and its predecessor are remapped (≈ 1/n of all keys)
- Removing a node: its keys go to the next node

Virtual nodes (multiple positions per physical node) smooth the distribution.

---

## Cryptographic Primitives

### Modular Exponentiation

The foundation of RSA, Diffie-Hellman, and digital signatures:

```
c = m^e mod n
```

Computed efficiently via **square-and-multiply** in O(log e) multiplications:

```python
def mod_pow(base, exp, mod):
    result = 1
    base = base % mod
    while exp > 0:
        if exp % 2 == 1:
            result = (result * base) % mod
        exp >>= 1
        base = (base * base) % mod
    return result
```

### Elliptic Curve Arithmetic

Elliptic curves over finite fields (y² = x³ + ax + b mod p) provide:
- **Smaller keys** for equivalent security (256-bit ECC ≈ 3072-bit RSA)
- **Faster operations** than RSA
- Foundation of ECDSA (Bitcoin, TLS), ECDH (key exchange)

### Hash-Based Structures

| Structure | Math | Application |
|-----------|------|-------------|
| Merkle tree | Hash(Hash(left) \|\| Hash(right)) | Git, blockchain, certificate transparency |
| Bloom filter | k hashes mod m bits | Set membership (probabilistic) |
| Count-min sketch | k hashes × counters | Frequency estimation (streaming) |
| HyperLogLog | Max of hash bit patterns | Cardinality estimation |

---

## Approximation and Estimation

### Fermi Estimation

Order-of-magnitude reasoning to quickly estimate quantities:

**How many requests per second does a service need to handle?**

```
1 million daily active users
× 10 actions per user per day
= 10 million requests per day
÷ 86,400 seconds per day
≈ 115 requests per second
× 10 for peak-to-average ratio
≈ 1,200 peak RPS
```

### Useful Approximations

| Expression | Approximation | When |
|-----------|---------------|------|
| (1 + x)ⁿ | 1 + nx | x is small |
| eˣ | 1 + x | x is small |
| ln(1 + x) | x | x is small |
| n! | √(2πn)(n/e)ⁿ | Stirling's approximation |
| C(n, k) | nᵏ/k! | k << n |
| Hₙ (harmonic) | ln(n) + 0.577 | |
| log₂(10) | 3.32 | Converting between bases |
| 2¹⁰ | 10³ | Quick binary/decimal |

### Little's Law

For any stable queuing system:

```
L = λW
```

Where L = average number of items in system, λ = arrival rate, W = average time in system.

**Example:** A service processes requests with λ = 100 req/s and average latency W = 50ms. Then L = 100 × 0.05 = 5 concurrent requests in the system at any time.

### Amdahl's Law

The theoretical speedup from parallelising a fraction f of a program with p processors:

```
Speedup = 1 / ((1 - f) + f/p)
```

If 90% of the work is parallelisable (f = 0.9) with 10 processors:

```
Speedup = 1 / (0.1 + 0.09) = 1/0.19 ≈ 5.3×
```

Not 10× — the serial 10% limits the speedup. This is why optimising the serial bottleneck matters more than adding processors.

---

## When to Reach for a Math Library

| Task | Library | Language |
|------|---------|---------|
| Linear algebra | NumPy, LAPACK, Eigen | Python, C/Fortran, C++ |
| Statistics | SciPy, R, pandas | Python, R |
| Symbolic math | SymPy, Mathematica | Python, Wolfram |
| Optimisation | SciPy.optimize, CVXPY | Python |
| Arbitrary precision | mpmath, GMP | Python, C |
| Probability distributions | SciPy.stats, NumPy.random | Python |
| Graph algorithms | NetworkX, igraph | Python, C |
| Cryptographic operations | hashlib, cryptography, OpenSSL | Python, C |

**Rule of thumb:** Never implement your own numerical algorithm when a well-tested library exists. The edge cases (overflow, underflow, denormals, NaN propagation, numerical stability) are where production bugs hide.

---

## Key Takeaways

- Numerical instability is a production bug category. Work in log-space, use compensated summation, and never compare floats for equality.
- Monte Carlo methods solve intractable problems through random sampling. The O(1/√N) convergence rate is dimension-independent — this is why they dominate in high dimensions.
- Hash function quality (uniformity, avalanche effect, collision resistance) is a mathematical property you can measure and verify.
- Cryptographic security reduces to hard mathematical problems (factoring, discrete logarithm, lattice problems). Key sizes are chosen based on the birthday bound and computational cost estimates.
- Fermi estimation, Little's Law, and Amdahl's Law are the three tools you will use most often in system design interviews and capacity planning.
- Use established math libraries. The numerical edge cases are harder than the algorithms themselves.
