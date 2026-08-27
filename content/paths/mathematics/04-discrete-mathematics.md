---
title: "Discrete Mathematics"
weight: 4
---

# Discrete Mathematics

Discrete mathematics deals with countable, separated values — the natural habitat of computers, which process distinct symbols, not continuous quantities. Sets, relations, and combinatorics form the theoretical foundation of data structures, databases, and algorithms.

---

## Sets

A **set** is an unordered collection of distinct elements.

### Notation

```
A = {1, 2, 3, 4, 5}           Roster notation
B = {x ∈ ℤ | x > 0 and x ≤ 5}  Set-builder notation
∅ = {}                          Empty set
```

### Set Operations

| Operation | Symbol | Definition | Example (A={1,2,3}, B={2,3,4}) |
|-----------|--------|------------|-------------------------------|
| Union | A ∪ B | Elements in A or B (or both) | {1, 2, 3, 4} |
| Intersection | A ∩ B | Elements in both A and B | {2, 3} |
| Difference | A \ B | Elements in A but not B | {1} |
| Symmetric difference | A △ B | Elements in exactly one of A or B | {1, 4} |
| Complement | Aᶜ | Elements not in A (relative to universe U) | U \ A |
| Cartesian product | A × B | All ordered pairs (a, b) | {(1,2),(1,3),(1,4),...} |

### Set Properties

| Property | Formula |
|----------|---------|
| Commutative | A ∪ B = B ∪ A, A ∩ B = B ∩ A |
| Associative | (A ∪ B) ∪ C = A ∪ (B ∪ C) |
| Distributive | A ∩ (B ∪ C) = (A ∩ B) ∪ (A ∩ C) |
| De Morgan's | (A ∪ B)ᶜ = Aᶜ ∩ Bᶜ, (A ∩ B)ᶜ = Aᶜ ∪ Bᶜ |
| Idempotent | A ∪ A = A, A ∩ A = A |
| Identity | A ∪ ∅ = A, A ∩ U = A |

### In Programming

```python
A = {1, 2, 3}
B = {2, 3, 4}

A | B      # {1, 2, 3, 4}  union
A & B      # {2, 3}         intersection
A - B      # {1}            difference
A ^ B      # {1, 4}         symmetric difference
A <= B     # False           subset check
```

SQL `SELECT` with `UNION`, `INTERSECT`, `EXCEPT` maps directly to set operations.

### Cardinality

The **cardinality** |A| of a set is the number of elements in it.

**Inclusion-Exclusion Principle:**

```
|A ∪ B| = |A| + |B| - |A ∩ B|
```

Generalised to three sets:

```
|A ∪ B ∪ C| = |A| + |B| + |C| - |A ∩ B| - |A ∩ C| - |B ∩ C| + |A ∩ B ∩ C|
```

**Application:** Counting unique visitors across multiple pages, deduplicating overlapping data sources.

### Power Set

The **power set** P(A) is the set of all subsets of A, including ∅ and A itself.

```
A = {1, 2, 3}
P(A) = {∅, {1}, {2}, {3}, {1,2}, {1,3}, {2,3}, {1,2,3}}
|P(A)| = 2^|A| = 2³ = 8
```

This is why iterating over all subsets is O(2ⁿ) — there are exactly 2ⁿ of them.

---

## Relations

A **relation** R from set A to set B is a subset of A × B. If (a, b) ∈ R, we write aRb.

### Properties of Relations on a Set

For a relation R on set A (i.e., R ⊂ A × A):

| Property | Definition | Example |
|----------|-----------|---------|
| Reflexive | aRa for all a ∈ A | ≤ (every number is ≤ itself) |
| Symmetric | aRb implies bRa | = (equality is symmetric) |
| Antisymmetric | aRb and bRa implies a = b | ≤ (if a ≤ b and b ≤ a, then a = b) |
| Transitive | aRb and bRc implies aRc | < (if a < b and b < c, then a < c) |

### Equivalence Relations

A relation that is reflexive, symmetric, and transitive. It partitions the set into **equivalence classes** — groups of elements that are "the same" under the relation.

| Relation | Equivalence Classes |
|----------|-------------------|
| Same remainder mod 3 | {0,3,6,...}, {1,4,7,...}, {2,5,8,...} |
| Same file extension | .py files, .java files, .go files |
| Same hash bucket | All keys with hash(k) % n = i |

### Partial Orders

A relation that is reflexive, antisymmetric, and transitive. Formalises "≤" and dependency relationships.

| Example | Domain |
|---------|--------|
| ≤ on integers | Number comparison |
| ⊆ on sets | Set containment |
| Task dependencies | Build systems, CI/CD pipelines |
| Version ordering | Semantic versioning |

A partial order where every pair of elements is comparable is a **total order** (like ≤ on integers). Otherwise, some elements are incomparable (like tasks with no dependency between them — they can run in parallel).

### Topological Sort

Given a partial order (a DAG), a topological sort produces a linear ordering consistent with the partial order. Every build system (Make, Gradle, Terraform) computes a topological sort of dependencies.

---

## Functions as Relations

A function f: A → B is a special relation where each element of A appears in exactly one pair.

### Types of Functions

| Type | Definition | Example |
|------|-----------|---------|
| Injective (one-to-one) | f(a) = f(b) implies a = b | Different inputs always give different outputs |
| Surjective (onto) | Every b ∈ B has some a with f(a) = b | Every output is hit |
| Bijective | Both injective and surjective | Perfect pairing — invertible |

### Engineering Applications

| Function Type | Application |
|--------------|-------------|
| Injective | Primary keys in databases (unique per row) |
| Surjective | Hash function onto bucket range (every bucket used) |
| Bijective | Encryption/decryption, encoding/decoding, reversible transformations |

---

## Combinatorics

Counting the number of ways to arrange, select, or combine elements. Essential for probability, algorithm analysis, and understanding search spaces.

### The Fundamental Counting Principle

If task 1 can be done in m ways and task 2 in n ways, then both tasks can be done in m × n ways (assuming independence).

```
Passwords with 8 lowercase letters: 26⁸ = 208,827,064,576
Passwords with 8 mixed-case + digits: 62⁸ = 218,340,105,584,896
```

### Permutations (Order Matters)

The number of ways to arrange k items chosen from n distinct items:

```
P(n, k) = n! / (n - k)!
```

| Scenario | Formula | Example |
|----------|---------|---------|
| Arrange all n items | n! | 5 books on a shelf: 5! = 120 |
| Choose and arrange k from n | n!/(n-k)! | Top 3 from 10 runners: 10×9×8 = 720 |
| Arrange with repetition | nᵏ | 4-digit PIN: 10⁴ = 10,000 |

### Combinations (Order Does Not Matter)

The number of ways to choose k items from n without regard to order:

```
C(n, k) = n! / (k! × (n - k)!)
```

Also written as $\binom{n}{k}$ ("n choose k").

| Scenario | Formula | Example |
|----------|---------|---------|
| Choose k from n | C(n,k) | Committee of 3 from 10: C(10,3) = 120 |
| Choose k or fewer from n | Σ C(n,i) for i=0 to k | |
| Choose any subset | 2ⁿ | All subsets of 10 items: 1024 |

### Key Identity: Pascal's Rule

```
C(n, k) = C(n-1, k-1) + C(n-1, k)
```

This recursive relation generates **Pascal's triangle** and is the basis for computing binomial coefficients efficiently.

### The Binomial Theorem

```
(a + b)ⁿ = Σ_{k=0}^{n} C(n,k) × aⁿ⁻ᵏ × bᵏ
```

Setting a = b = 1: Σ C(n,k) = 2ⁿ (total number of subsets).

### Stars and Bars

The number of ways to put n identical items into k distinct bins:

```
C(n + k - 1, k - 1)
```

**Application:** Distributing 10 identical tasks across 3 servers = C(12, 2) = 66 ways.

---

## The Pigeonhole Principle

If n items are placed into m containers and n > m, at least one container has more than one item.

### Generalised Form

If n items go into m containers, at least one container has ⌈n/m⌉ items.

### Applications

| Problem | Pigeonhole Argument |
|---------|-------------------|
| Hash collisions | 2⁶⁴ possible inputs into 2³² buckets → collisions guaranteed |
| Birthday paradox setup | 367 people → at least 2 share a birthday (366 possible days + leap) |
| Lossless compression | Not all files can be compressed (2ⁿ files can't all map to fewer than 2ⁿ outputs) |
| Network routing | n+1 packets through n links → at least one link carries 2+ packets |

---

## Recurrence Relations

An equation that defines a sequence where each term depends on previous terms. Fundamental for analysing recursive algorithms.

### Common Recurrences in CS

| Recurrence | Closed Form | Algorithm |
|-----------|-------------|-----------|
| T(n) = T(n-1) + 1 | O(n) | Linear scan/recursion |
| T(n) = T(n-1) + n | O(n²) | Selection sort |
| T(n) = 2T(n/2) + n | O(n log n) | Merge sort |
| T(n) = 2T(n/2) + 1 | O(n) | Binary tree traversal |
| T(n) = T(n/2) + 1 | O(log n) | Binary search |
| T(n) = 2T(n-1) + 1 | O(2ⁿ) | Tower of Hanoi |
| T(n) = T(n-1) + T(n-2) | O(φⁿ) | Fibonacci (naive) |

### Solving Recurrences

**1. Substitution method:** Guess the form, prove by induction.

**2. Recursion tree:** Visualise the work at each level.

```
T(n) = 2T(n/2) + n

Level 0:               n           work = n
Level 1:         n/2     n/2       work = n
Level 2:       n/4 n/4 n/4 n/4    work = n
  ...
Level log₂n:  1 1 1 ... 1         work = n

Total: n × log₂(n) levels = O(n log n)
```

**3. Master theorem** for recurrences of the form T(n) = aT(n/b) + O(nᵈ):

| Condition | Result |
|-----------|--------|
| d > log_b(a) | T(n) = O(nᵈ) |
| d = log_b(a) | T(n) = O(nᵈ log n) |
| d < log_b(a) | T(n) = O(n^(log_b(a))) |

**Example:** Merge sort: T(n) = 2T(n/2) + O(n). Here a=2, b=2, d=1. Since d = log₂(2) = 1, we get T(n) = O(n log n). ✓

---

## Modular Arithmetic Revisited

In the context of discrete math, we can define the set of integers modulo n:

```
ℤ_n = {0, 1, 2, ..., n-1}
```

This forms a **ring** under addition and multiplication modulo n. When n is prime, it forms a **field** — every nonzero element has a multiplicative inverse. This is the foundation of finite field cryptography (AES, elliptic curves).

### Fermat's Little Theorem

If p is prime and a is not divisible by p:

```
a^(p-1) ≡ 1 (mod p)
```

This is used to compute modular inverses: a⁻¹ ≡ a^(p-2) (mod p).

### Applications

| Application | Mathematical Foundation |
|-------------|----------------------|
| RSA encryption | Euler's theorem in ℤ_n |
| Diffie-Hellman key exchange | Discrete logarithm in ℤ_p* |
| Error-detecting codes | Polynomial arithmetic in GF(2) |
| Hash functions | Modular arithmetic in ℤ_p |

---

## Key Takeaways

- Sets and their operations are the mathematical model behind database queries, type systems, and access control.
- Relations formalise dependencies — equivalence relations partition data, partial orders enable topological sort.
- Combinatorics (permutations, combinations, pigeonhole) is essential for understanding search spaces, password strength, and hash collision probability.
- Recurrence relations describe recursive algorithms. The Master theorem lets you solve them mechanically.
- Modular arithmetic on prime fields is the foundation of modern cryptography.
