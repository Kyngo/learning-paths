---
title: "Logic & Proofs"
weight: 5
---

# Logic & Proofs

Logic is the science of valid reasoning — the same principles that govern circuit design, type systems, query optimisation, and formal verification. Proof techniques are how you demonstrate that algorithms are correct and that security properties hold.

---

## Propositional Logic

Propositional logic deals with statements (propositions) that are either true or false, connected by logical operators.

### Logical Connectives

| Connective | Symbol | Name | English | Example |
|-----------|--------|------|---------|---------|
| NOT | ¬ | Negation | "not p" | ¬(it is raining) |
| AND | ∧ | Conjunction | "p and q" | (sunny) ∧ (warm) |
| OR | ∨ | Disjunction | "p or q" (inclusive) | (admin) ∨ (owner) |
| XOR | ⊕ | Exclusive or | "p or q but not both" | (left) ⊕ (right) |
| IMPLIES | → | Implication | "if p then q" | (logged_in) → (has_session) |
| IFF | ↔ | Biconditional | "p if and only if q" | (x>0) ↔ (x is positive) |

### Truth Tables

**AND:**

| p | q | p ∧ q |
|---|---|-------|
| T | T | T |
| T | F | F |
| F | T | F |
| F | F | F |

**OR:**

| p | q | p ∨ q |
|---|---|-------|
| T | T | T |
| T | F | T |
| F | T | T |
| F | F | F |

**IMPLIES (the tricky one):**

| p | q | p → q |
|---|---|-------|
| T | T | T |
| T | F | F |
| F | T | T |
| F | F | T |

The implication "if p then q" is only false when p is true and q is false. When p is false, the implication is **vacuously true** — "if pigs can fly, then I am the queen" is technically true because pigs cannot fly.

### In Programming

| Logic | Code |
|-------|------|
| ¬p | `not p` / `!p` |
| p ∧ q | `p and q` / `p && q` |
| p ∨ q | `p or q` / `p \|\| q` |
| p → q | `not p or q` / `if p then q` |
| p ⊕ q | `p ^ q` (bitwise) / `p != q` (boolean) |

### Short-Circuit Evaluation

Most languages evaluate `&&` and `||` left-to-right and stop early:

```python
# Safe because of short-circuit
if x != 0 and y / x > threshold:
    ...

# Second condition is never evaluated when x == 0
```

This is a direct application of the truth tables: `false ∧ anything = false`, so the second operand is unnecessary.

---

## Logical Equivalences

Two propositions are **logically equivalent** (≡) if they have identical truth tables.

### Key Equivalences

| Name | Equivalence |
|------|-------------|
| Double negation | ¬(¬p) ≡ p |
| De Morgan's laws | ¬(p ∧ q) ≡ ¬p ∨ ¬q |
| | ¬(p ∨ q) ≡ ¬p ∧ ¬q |
| Contrapositive | (p → q) ≡ (¬q → ¬p) |
| Implication | (p → q) ≡ (¬p ∨ q) |
| Distributive | p ∧ (q ∨ r) ≡ (p ∧ q) ∨ (p ∧ r) |
| | p ∨ (q ∧ r) ≡ (p ∨ q) ∧ (p ∨ r) |
| Absorption | p ∧ (p ∨ q) ≡ p |
| | p ∨ (p ∧ q) ≡ p |

### De Morgan's Laws in Code

```python
# These are equivalent:
not (a and b)  ==  (not a) or (not b)
not (a or b)   ==  (not a) and (not b)

# Refactoring complex conditions:
# Original:
if not (user.is_admin and user.is_active):
    deny()

# De Morgan's:
if not user.is_admin or not user.is_active:
    deny()
```

### Contrapositive in Debugging

The contrapositive of "if p then q" is "if not q then not p" — they are logically equivalent.

```
Statement:     "If the service is healthy, the health check returns 200."
Contrapositive: "If the health check does NOT return 200, the service is NOT healthy."
```

The contrapositive is often easier to test than the original statement.

---

## Predicate Logic

Extends propositional logic with **variables** and **quantifiers**, allowing statements about collections of objects.

### Quantifiers

| Quantifier | Symbol | Meaning | Example |
|-----------|--------|---------|---------|
| Universal | ∀ | "for all" | ∀x ∈ users: x.age ≥ 0 |
| Existential | ∃ | "there exists" | ∃x ∈ users: x.role = "admin" |

### Negating Quantifiers

```
¬(∀x: P(x))  ≡  ∃x: ¬P(x)
¬(∃x: P(x))  ≡  ∀x: ¬P(x)
```

In English:
- "Not all users are active" = "There exists a user who is not active"
- "No user is admin" = "For all users, the user is not admin"

### In Programming and SQL

```sql
-- ∀x ∈ orders: x.total > 0
-- "All orders have positive totals"
SELECT NOT EXISTS (SELECT 1 FROM orders WHERE total <= 0);

-- ∃x ∈ orders: x.status = 'refunded'
-- "There exists a refunded order"
SELECT EXISTS (SELECT 1 FROM orders WHERE status = 'refunded');
```

```python
# ∀x: P(x)
all(user.age >= 0 for user in users)

# ∃x: P(x)
any(user.role == "admin" for user in users)
```

---

## Boolean Algebra

Boolean algebra is propositional logic applied to the values {0, 1} — the foundation of digital circuit design and bitwise operations.

### Axioms

| Axiom | AND form | OR form |
|-------|----------|---------|
| Identity | a ∧ 1 = a | a ∨ 0 = a |
| Null | a ∧ 0 = 0 | a ∨ 1 = 1 |
| Complement | a ∧ ¬a = 0 | a ∨ ¬a = 1 |
| Idempotent | a ∧ a = a | a ∨ a = a |

### Normal Forms

Any Boolean expression can be written in:

- **CNF (Conjunctive Normal Form):** AND of ORs — `(a ∨ b) ∧ (¬a ∨ c) ∧ (b ∨ ¬c)`
- **DNF (Disjunctive Normal Form):** OR of ANDs — `(a ∧ b) ∨ (¬a ∧ c) ∨ (b ∧ ¬c)`

**Relevance:** SAT solvers (used in verification, scheduling, package managers) work on CNF. WHERE clauses in SQL are Boolean expressions that query optimisers convert to normal forms.

---

## Proof Techniques

### 1. Direct Proof

Assume the hypothesis and derive the conclusion through logical steps.

**Claim:** If n is even, then n² is even.

**Proof:** n is even → n = 2k for some integer k → n² = (2k)² = 4k² = 2(2k²) → n² is even. ∎

### 2. Proof by Contrapositive

Prove ¬q → ¬p instead of p → q (they are logically equivalent).

**Claim:** If n² is odd, then n is odd.

**Proof** (contrapositive: if n is even, then n² is even): n even → n = 2k → n² = 4k² = 2(2k²) → n² is even. ∎

### 3. Proof by Contradiction

Assume the statement is false and derive a contradiction.

**Claim:** √2 is irrational.

**Proof:** Assume √2 = p/q in lowest terms (gcd(p,q) = 1).
Then 2 = p²/q², so p² = 2q². This means p² is even, so p is even (by contrapositive above).
Let p = 2k. Then (2k)² = 2q² → 4k² = 2q² → q² = 2k² → q is even.
But if both p and q are even, gcd(p,q) ≥ 2, contradicting our assumption. ∎

### 4. Mathematical Induction

Prove a statement P(n) for all n ≥ n₀:

1. **Base case:** Prove P(n₀)
2. **Inductive step:** Prove P(k) → P(k+1) for arbitrary k ≥ n₀

**Claim:** Σᵢ₌₁ⁿ i = n(n+1)/2

**Base case:** P(1): Σᵢ₌₁¹ i = 1 = 1(2)/2 ✓

**Inductive step:** Assume Σᵢ₌₁ᵏ i = k(k+1)/2. Then:

```
Σᵢ₌₁^(k+1) i = Σᵢ₌₁ᵏ i + (k+1)
              = k(k+1)/2 + (k+1)
              = (k+1)(k/2 + 1)
              = (k+1)(k+2)/2  ✓
```

### 5. Strong Induction

Like regular induction, but the inductive hypothesis assumes P(n₀), P(n₀+1), ..., P(k) are all true, not just P(k).

Useful when P(k+1) depends on multiple previous cases (e.g., Fibonacci, optimal substructure in dynamic programming).

### 6. Proof by Construction

Prove existence by explicitly building the object.

**Claim:** For any finite set S, a bijection exists between S and {1, 2, ..., |S|}.

**Proof:** List the elements of S in any order as s₁, s₂, ..., s_{|S|}. Map sᵢ → i. This is a bijection. ∎

### Which Proof to Use When

| Situation | Technique |
|-----------|-----------|
| "If A then B" | Direct proof or contrapositive |
| "A if and only if B" | Prove both directions |
| "X does not exist" or "X is impossible" | Contradiction |
| Statement about all n ≥ n₀ | Induction |
| "There exists an X" | Construction or contradiction |

---

## Logic in Systems

### Type Systems as Logic

The **Curry-Howard correspondence** maps:

| Logic | Types |
|-------|-------|
| Proposition | Type |
| Proof | Program |
| Implication (A → B) | Function type (A → B) |
| Conjunction (A ∧ B) | Product type (A, B) / tuple / struct |
| Disjunction (A ∨ B) | Sum type (Either A B) / enum |
| True | Unit type () |
| False | Empty type (Void / Never) |

A well-typed program is a proof that the type is inhabited. This is why type systems can catch errors at compile time — they are performing logical reasoning.

### Invariants and Loop Correctness

A **loop invariant** is a property that is true before and after each iteration:

```
# Invariant: sum == Σ arr[0..i-1]
sum = 0
for i in range(len(arr)):
    # Invariant holds here (before body)
    sum += arr[i]
    # Invariant holds here (after body)
# After loop: sum == Σ arr[0..n-1] (total sum)
```

Proving loop correctness = induction where the base case is initialisation, the inductive step is one iteration, and the postcondition follows from the invariant plus the loop termination condition.

---

## Key Takeaways

- Propositional logic (AND, OR, NOT, IMPLIES) is the foundation of conditional expressions, circuit design, and query optimisation.
- De Morgan's laws and the contrapositive are the two most practically useful logical equivalences for code refactoring and debugging.
- Predicate logic (∀, ∃) formalises the reasoning behind SQL queries, type constraints, and specification languages.
- Induction is the proof technique for recursive algorithms. If you can express the algorithm as a recurrence, you can prove it correct by induction.
- Boolean algebra (CNF, DNF) underpins SAT solvers, package dependency resolution, and hardware synthesis.
- The Curry-Howard correspondence means that type-checking IS proof-checking — a program that compiles is a proof that the types are consistent.
