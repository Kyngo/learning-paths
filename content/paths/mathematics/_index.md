---
title: "Mathematics for Engineers"
weight: 15
bookCollapseSection: true
---

# Mathematics for Engineers

The mathematical foundations that underpin computer science, machine learning, cryptography, and everyday engineering decisions — from number systems to information theory.

## Overview

Mathematics is not a separate discipline from engineering — it is the language in which engineering problems are precisely stated and solved. This path covers the mathematics that working software engineers actually use: the number theory behind hashing and cryptography, the linear algebra behind ML embeddings and graphics, the probability behind A/B tests and system reliability, and the discrete structures behind every algorithm and data structure.

The approach is practical. Every concept is motivated by a real engineering application before the formalism is introduced. Proofs are included where they build intuition, not for their own sake.

## Prerequisites

- Basic arithmetic (you can do this)
- Some programming experience (examples use Python-style pseudocode)
- Willingness to work through problems — mathematics is learned by doing, not reading

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Numbers & Arithmetic]({{< relref "01-numbers-and-arithmetic" >}}) | Number systems, binary/hex/octal, modular arithmetic, floating-point |
| 2 | [Algebra]({{< relref "02-algebra" >}}) | Expressions, equations, logarithms, exponents, summation notation |
| 3 | [Functions & Graphs]({{< relref "03-functions-and-graphs" >}}) | Domain/range, composition, inverse, growth rates, floor/ceil |
| 4 | [Discrete Mathematics]({{< relref "04-discrete-mathematics" >}}) | Sets, relations, combinatorics, recurrence relations, counting |
| 5 | [Logic & Proofs]({{< relref "05-logic-and-proofs" >}}) | Propositional/predicate logic, truth tables, proof techniques, Boolean algebra |
| 6 | [Graph Theory]({{< relref "06-graph-theory" >}}) | Graphs, trees, traversals, shortest paths, spanning trees, network flow |
| 7 | [Linear Algebra]({{< relref "07-linear-algebra" >}}) | Vectors, matrices, determinants, eigenvalues, transformations |
| 8 | [Probability]({{< relref "08-probability" >}}) | Sample spaces, Bayes' theorem, distributions, expected value |
| 9 | [Statistics]({{< relref "09-statistics" >}}) | Descriptive stats, hypothesis testing, regression, A/B testing |
| 10 | [Calculus Essentials]({{< relref "10-calculus-essentials" >}}) | Limits, derivatives, integrals, gradient descent intuition |
| 11 | [Information Theory]({{< relref "11-information-theory" >}}) | Entropy, cross-entropy, KL divergence, compression, error correction |
| 12 | [Mathematics in Practice]({{< relref "12-math-in-practice" >}}) | Numerical stability, floating-point traps, Monte Carlo, hashing math |

## Learning Approach

Each section includes:

- **Motivation** — why this topic matters for engineers, with concrete applications
- **Formal definitions** — precise mathematical language, introduced gently
- **Worked examples** — step-by-step solutions with engineering context
- **Tables and visual aids** — reference material for quick lookup
- **Key takeaways** — the essential ideas to carry forward

## How the Sections Connect

```text
Numbers & Arithmetic ──→ Algebra ──→ Functions & Graphs
        │                                    │
        ▼                                    ▼
Discrete Mathematics ──→ Logic & Proofs ──→ Graph Theory
        │                    │
        ▼                    ▼
Linear Algebra         Probability ──→ Statistics
        │                    │
        ▼                    ▼
Calculus Essentials    Information Theory
        │                    │
        └────────┬───────────┘
                 ▼
      Mathematics in Practice
```

## Recommended Pairings

This path complements several others in the collection:

- **Programming Logic** — algorithms and data structures use discrete math and graph theory daily
- **DSA Interview Prep** — combinatorics and recurrence relations explain why algorithms work
- **AI & Large Language Models** — linear algebra, probability, calculus, and information theory are the mathematical backbone of ML
- **System Design** — probability and statistics inform capacity planning, SLO budgets, and load balancing
- **Cybersecurity** — number theory and information theory underpin cryptography
