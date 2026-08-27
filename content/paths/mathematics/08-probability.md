---
title: "Probability"
weight: 8
---

# Probability

Probability quantifies uncertainty. It is the mathematical framework behind reliability engineering, machine learning, A/B testing, cryptographic security, and capacity planning. If you build systems that interact with the real world, you reason about probability whether you formalise it or not.

---

## Foundations

### Sample Space and Events

- **Experiment:** A process with uncertain outcome (rolling a die, receiving a request)
- **Sample space (Ω):** The set of all possible outcomes
- **Event (A):** A subset of the sample space

```
Rolling a die:
  Ω = {1, 2, 3, 4, 5, 6}
  A = "even number" = {2, 4, 6}
  P(A) = |A| / |Ω| = 3/6 = 1/2
```

### Axioms of Probability

1. P(A) ≥ 0 for any event A
2. P(Ω) = 1 (something must happen)
3. For mutually exclusive events: P(A ∪ B) = P(A) + P(B)

### Basic Rules

| Rule | Formula |
|------|---------|
| Complement | P(not A) = 1 - P(A) |
| Union | P(A ∪ B) = P(A) + P(B) - P(A ∩ B) |
| Mutually exclusive | P(A ∪ B) = P(A) + P(B) |
| Certain event | P(Ω) = 1 |
| Impossible event | P(∅) = 0 |

---

## Conditional Probability

The probability of A given that B has occurred:

```
P(A | B) = P(A ∩ B) / P(B)
```

**Example:** A service has 99.9% uptime. Given that it was up at 9 AM, what is the probability it is still up at 10 AM? This depends on the failure model (independent failures vs. correlated).

### Multiplication Rule

```
P(A ∩ B) = P(A | B) × P(B) = P(B | A) × P(A)
```

### Independence

Events A and B are **independent** if knowing one gives no information about the other:

```
P(A ∩ B) = P(A) × P(B)
P(A | B) = P(A)
```

**Warning:** Independence is an assumption, not a default. Network failures are often correlated (same switch, same datacenter, same ISP).

---

## Bayes' Theorem

Bayes' theorem inverts conditional probability — it lets you update beliefs when new evidence arrives:

```
P(A | B) = P(B | A) × P(A) / P(B)
```

| Term | Name | Meaning |
|------|------|---------|
| P(A) | Prior | Initial belief before evidence |
| P(B \| A) | Likelihood | How likely the evidence is if A is true |
| P(A \| B) | Posterior | Updated belief after seeing evidence |
| P(B) | Evidence | Total probability of the evidence |

### Example: Diagnostic Testing

A disease affects 1% of the population. A test has:
- 99% sensitivity (true positive rate): P(positive | disease) = 0.99
- 95% specificity (true negative rate): P(negative | healthy) = 0.95

If you test positive, what is the probability you have the disease?

```
P(disease | positive) = P(positive | disease) × P(disease) / P(positive)

P(positive) = P(positive | disease) × P(disease) + P(positive | healthy) × P(healthy)
            = 0.99 × 0.01 + 0.05 × 0.99
            = 0.0099 + 0.0495
            = 0.0594

P(disease | positive) = 0.0099 / 0.0594 ≈ 16.7%
```

Despite a 99% accurate test, a positive result only means ~17% chance of disease. This is because the disease is rare (low prior), so false positives dominate.

### Engineering Applications of Bayes

| Application | Prior | Evidence | Posterior |
|-------------|-------|----------|-----------|
| Spam filtering | P(spam) | Words in email | P(spam \| words) |
| Anomaly detection | P(attack) | Log patterns | P(attack \| patterns) |
| A/B testing (Bayesian) | P(variant better) | Conversion data | Updated P(variant better) |
| Alert fatigue | P(real incident) | Alert fired | P(real \| alert) |
| Medical AI | Disease prevalence | Scan features | Diagnosis confidence |

---

## Random Variables and Distributions

A **random variable** X assigns a numerical value to each outcome. A **probability distribution** describes the probabilities of all possible values.

### Discrete Distributions

| Distribution | Parameters | PMF | E[X] | Var(X) | Use |
|-------------|-----------|-----|------|--------|-----|
| Bernoulli | p | P(X=1)=p, P(X=0)=1-p | p | p(1-p) | Single yes/no trial |
| Binomial | n, p | C(n,k)pᵏ(1-p)ⁿ⁻ᵏ | np | np(1-p) | Number of successes in n trials |
| Geometric | p | p(1-p)ᵏ⁻¹ | 1/p | (1-p)/p² | Trials until first success |
| Poisson | λ | e⁻λ λᵏ/k! | λ | λ | Events per time interval |

**Poisson in engineering:** Requests per second to a server, errors per day, arrivals in a queue. If events occur independently at a constant average rate λ, they follow a Poisson distribution.

### Continuous Distributions

| Distribution | Parameters | PDF | E[X] | Var(X) | Use |
|-------------|-----------|-----|------|--------|-----|
| Uniform | a, b | 1/(b-a) | (a+b)/2 | (b-a)²/12 | Equal probability over range |
| Exponential | λ | λe⁻λˣ | 1/λ | 1/λ² | Time between events |
| Normal (Gaussian) | μ, σ² | (1/σ√2π)e^(-(x-μ)²/2σ²) | μ | σ² | Natural variation, errors |

### The Normal Distribution

The most important distribution in statistics. Properties:

- Bell-shaped, symmetric around the mean μ
- 68% of values within μ ± σ
- 95% within μ ± 2σ
- 99.7% within μ ± 3σ

**The 68-95-99.7 rule** is used constantly: SLOs, quality control, performance thresholds.

### Central Limit Theorem (CLT)

The average of many independent random variables tends toward a normal distribution, regardless of the underlying distribution, as the sample size grows.

This is why the normal distribution appears everywhere — any measurement that is the sum of many small independent effects will be approximately normal.

---

## Expected Value and Variance

### Expected Value (Mean)

The long-run average:

```
E[X] = Σ xᵢ P(X = xᵢ)          (discrete)
E[X] = ∫ x f(x) dx              (continuous)
```

**Properties:**
- E[aX + b] = aE[X] + b (linearity)
- E[X + Y] = E[X] + E[Y] (always, even if dependent)
- E[XY] = E[X]E[Y] only if X and Y are independent

**Application:** Expected cost of a retry strategy. If each retry has P(success) = p and cost c:

```
E[total cost] = c/p    (geometric distribution)
```

### Variance and Standard Deviation

Variance measures spread:

```
Var(X) = E[(X - E[X])²] = E[X²] - (E[X])²
```

Standard deviation: σ = √Var(X)

**Properties:**
- Var(aX + b) = a²Var(X)
- Var(X + Y) = Var(X) + Var(Y) if X, Y independent

---

## Important Results

### Law of Large Numbers

As the number of trials increases, the sample average converges to the expected value. This is why Monte Carlo simulation works.

### Birthday Paradox

In a group of n people, the probability that at least two share a birthday reaches 50% at just n = 23.

```
P(collision) ≈ 1 - e^(-n²/(2×365))
```

Generalised: in a space of d possible values, expect a collision after about √(πd/2) random samples.

| Space Size | Collision after ~√d samples | Application |
|-----------|---------------------------|-------------|
| 365 | ~23 | Birthdays |
| 2³² | ~77,000 | 32-bit hash |
| 2⁶⁴ | ~5 × 10⁹ | 64-bit hash |
| 2¹²⁸ | ~1.8 × 10¹⁹ | 128-bit hash (safe) |
| 2²⁵⁶ | ~3.4 × 10³⁸ | 256-bit hash (very safe) |

This is why 32-bit hashes are inadequate for large datasets and why cryptographic hashes use 256+ bits.

### Coupon Collector Problem

To collect all n distinct coupons (with random draws, uniform distribution), expected draws needed:

```
E[draws] = n × Hₙ ≈ n × ln(n) + 0.577n
```

Where Hₙ is the nth harmonic number. For n = 100, expect ~518 draws.

**Application:** Expected time to discover all endpoints in a fuzz test, expected requests before all cache slots are populated.

---

## Probability in System Design

### Availability Composition

| Configuration | Formula | Example (each 99%) |
|--------------|---------|-------------------|
| Series (all must work) | A₁ × A₂ × ... × Aₙ | 0.99³ = 97.03% |
| Parallel (any one works) | 1 - (1-A₁)(1-A₂)...(1-Aₙ) | 1 - 0.01³ = 99.9999% |

**Lesson:** Redundancy (parallel) dramatically improves availability. Serial dependencies (each service depends on the next) multiply failure probabilities.

### Consistent Hashing

A ring of 2³² positions. Each node owns a segment. The probability that adding a node affects a given key is 1/n (where n is the number of nodes), ensuring minimal redistribution.

### Bloom Filters

A probabilistic data structure with:
- False positives: probability ≈ (1 - e^(-kn/m))^k
- Zero false negatives
- Optimal k = (m/n) × ln(2) hash functions

Where m = filter size (bits), n = elements, k = hash functions.

---

## Key Takeaways

- Probability quantifies uncertainty. P(complement) = 1 - P(event) is the most useful identity.
- Bayes' theorem updates beliefs with evidence. The base rate (prior) matters enormously — rare conditions produce many false positives even with accurate tests.
- The Poisson distribution models arrival rates (requests/sec, errors/day). The exponential distribution models time between arrivals.
- The normal distribution appears everywhere because of the Central Limit Theorem — averages of independent variables converge to it.
- The birthday paradox governs hash collision probability: expect collisions after ~√d random values in a d-size space.
- Availability in series multiplies; in parallel it complements — this is the mathematical argument for redundancy.
