---
title: "Statistics"
weight: 9
---

# Statistics

Statistics is the discipline of learning from data — summarising it, drawing inferences, and making decisions under uncertainty. For engineers, it is the foundation of A/B testing, performance benchmarking, capacity planning, and data-driven decision-making.

---

## Descriptive Statistics

### Measures of Central Tendency

| Measure | Definition | When to Use | Sensitive to Outliers |
|---------|-----------|-------------|----------------------|
| Mean | Sum / count | Symmetric data | Yes |
| Median | Middle value (sorted) | Skewed data, outliers | No |
| Mode | Most frequent value | Categorical data | No |

**Example:** Response times [12, 15, 14, 13, 850] ms

- Mean = 180.8 ms (misleading — one outlier inflates it)
- Median = 14 ms (representative)
- P99 = 850 ms (captures tail behaviour)

### Measures of Spread

| Measure | Formula | Meaning |
|---------|---------|---------|
| Range | max - min | Total spread |
| Variance (σ²) | Σ(xᵢ - x̄)² / n | Average squared deviation |
| Standard deviation (σ) | √variance | Spread in original units |
| Interquartile range (IQR) | Q3 - Q1 | Spread of middle 50% |

### Percentiles

The pth percentile is the value below which p% of observations fall.

| Percentile | Name | SRE Use |
|-----------|------|---------|
| P50 | Median | Typical experience |
| P90 | 90th percentile | Most users |
| P95 | 95th percentile | Commonly used SLI |
| P99 | 99th percentile | Tail latency |
| P99.9 | 99.9th percentile | Worst-case users |

**Key insight:** Averages hide tail behaviour. If your SLO is P99 < 200ms, an average of 50ms tells you nothing about whether you are meeting it.

---

## Data Distributions in Practice

### Recognising Distributions

| Shape | Characteristics | Common In |
|-------|----------------|-----------|
| Normal (bell curve) | Symmetric, mean = median | Height, IQ, measurement errors |
| Right-skewed | Long right tail, mean > median | Response times, salaries, file sizes |
| Left-skewed | Long left tail, mean < median | Age at retirement, test scores with ceiling |
| Bimodal | Two peaks | Mixed populations (two server types) |
| Power law | Linear on log-log plot | Web page popularity, word frequencies |

### Log-Normal Distribution

If log(X) is normally distributed, X is log-normal. This models many engineering quantities:

- Response times
- File sizes
- User session lengths
- Error counts

**Practical consequence:** Use geometric mean or median for log-normal data, not arithmetic mean. Log-transform before applying methods that assume normality.

---

## Inferential Statistics

### Sampling

| Concept | Definition |
|---------|-----------|
| Population | The complete set of interest |
| Sample | A subset used to estimate population properties |
| Sampling bias | Systematic error from non-representative samples |
| Standard error | σ / √n — how much the sample mean varies |

The standard error decreases with √n: to halve the uncertainty, quadruple the sample size.

### Confidence Intervals

A range that contains the true parameter with a specified probability.

```
95% CI for mean = x̄ ± 1.96 × (s / √n)
```

| Confidence Level | z-score | Interpretation |
|-----------------|---------|----------------|
| 90% | 1.645 | Wider range, more confident |
| 95% | 1.960 | Standard choice |
| 99% | 2.576 | Widest range, most conservative |

**Common mistake:** "95% CI" does NOT mean "95% probability the true value is in this interval." It means: if you repeated the experiment many times, 95% of the CIs would contain the true value.

---

## Hypothesis Testing

A structured framework for deciding whether observed data is consistent with a null hypothesis.

### The Process

1. **State hypotheses:**
   - H₀ (null): no effect, no difference (the default)
   - H₁ (alternative): there is an effect or difference

2. **Choose significance level α** (usually 0.05)

3. **Compute test statistic** from the data

4. **Calculate p-value:** probability of seeing results this extreme (or more) under H₀

5. **Decision:**
   - p ≤ α → reject H₀ (statistically significant)
   - p > α → fail to reject H₀ (not enough evidence)

### Error Types

| | H₀ is true | H₀ is false |
|---|-----------|-------------|
| Reject H₀ | **Type I error** (false positive), probability α | ✓ Correct (power) |
| Fail to reject H₀ | ✓ Correct | **Type II error** (false negative), probability β |

**Statistical power** = 1 - β = probability of detecting a real effect.

### Common Tests

| Situation | Test | When |
|-----------|------|------|
| Compare mean to a value | One-sample t-test | "Is mean latency different from 100ms?" |
| Compare two group means | Two-sample t-test | "Is variant A faster than variant B?" |
| Compare proportions | Chi-squared test | "Is conversion rate different between groups?" |
| Compare distributions | Kolmogorov-Smirnov | "Do these two samples follow the same distribution?" |
| Non-parametric comparison | Mann-Whitney U | "Is one group consistently higher?" (no normality assumption) |

### P-Value Misinterpretations

| Wrong | Right |
|-------|-------|
| "p = 0.03 means 3% chance the null is true" | "If H₀ were true, there's a 3% chance of seeing data this extreme" |
| "p > 0.05 means H₀ is proven" | "We didn't find enough evidence to reject H₀" |
| "Significant = important" | "Significant = unlikely under H₀" — effect size matters too |
| "Lower p = bigger effect" | p depends on sample size; effect size is separate |

---

## A/B Testing

The controlled experiment framework used in product engineering.

### Design

```
Control (A):  Current version    → measure outcome
Treatment (B): Modified version  → measure outcome
```

### Sample Size Calculation

To detect a minimum detectable effect (MDE) with power 1-β and significance α:

```
n ≈ (z_α/2 + z_β)² × 2σ² / MDE²
```

| Desired | Consequence |
|---------|-------------|
| Smaller MDE | Need more samples |
| Higher confidence (lower α) | Need more samples |
| Higher power (lower β) | Need more samples |
| Higher variance | Need more samples |

**Rule of thumb:** Halving the MDE requires 4× the sample size.

### Common Pitfalls

| Pitfall | Problem | Mitigation |
|---------|---------|------------|
| Peeking | Checking results repeatedly inflates false positive rate | Pre-commit to analysis time or use sequential testing |
| Sample ratio mismatch | Unequal split signals a bug | Check traffic allocation before analysing results |
| Simpson's paradox | Aggregate results contradict subgroup results | Segment by key dimensions |
| Novelty/primacy effects | Users behave differently just because something changed | Run longer, use holdout groups |
| Multiple comparisons | Testing 20 variants → expect 1 false positive at α=0.05 | Bonferroni correction: α' = α / k |

---

## Regression

### Linear Regression

Models the relationship between a dependent variable y and independent variable(s) x:

```
y = β₀ + β₁x₁ + β₂x₂ + ... + ε
```

Where β₀ is the intercept, β₁...βₙ are coefficients, and ε is the error term.

### Ordinary Least Squares (OLS)

Finds coefficients that minimise the sum of squared residuals:

```
min Σ (yᵢ - ŷᵢ)²
```

### Evaluating Fit

| Metric | Formula | Meaning |
|--------|---------|---------|
| R² | 1 - (SS_res / SS_tot) | Proportion of variance explained (0 to 1) |
| RMSE | √(Σ(yᵢ - ŷᵢ)²/n) | Average prediction error in original units |
| MAE | Σ\|yᵢ - ŷᵢ\|/n | Average absolute error |

### Interpreting Coefficients

β₁ = 2.5 means: for each unit increase in x₁, y increases by 2.5 on average, holding other variables constant.

### Correlation vs. Causation

**Correlation** (r) measures linear association:
- r = +1: perfect positive linear
- r = 0: no linear relationship
- r = -1: perfect negative linear

```
r = Σ(xᵢ - x̄)(yᵢ - ȳ) / √(Σ(xᵢ - x̄)² × Σ(yᵢ - ȳ)²)
```

**Correlation does not imply causation.** Ice cream sales and drowning deaths are correlated (both increase in summer) but one does not cause the other.

To establish causation, you need:
1. Controlled experiments (A/B tests)
2. Natural experiments
3. Instrumental variables
4. Or very careful observational study with domain knowledge

---

## Practical Statistics for Engineers

### SLO Math

If your SLO is 99.9% availability per month:

```
Error budget = 1 - 0.999 = 0.001
Minutes in a month ≈ 43,200
Allowed downtime = 43,200 × 0.001 = 43.2 minutes/month
```

### Capacity Planning with Percentiles

If P99 latency = 200ms and you have 1000 req/s:
- 10 requests per second experience ≥ 200ms
- Over 24 hours: 864,000 slow requests

### Anomaly Detection

Standard approach: flag values outside μ ± kσ

| k | % flagged (normal data) | Use |
|---|------------------------|-----|
| 2 | 4.6% | Loose threshold |
| 3 | 0.3% | Standard anomaly detection |
| 4 | 0.006% | Conservative |

For non-normal data, use MAD (Median Absolute Deviation) instead of standard deviation.

### The Bootstrap

A resampling technique: sample with replacement from your data to estimate the distribution of any statistic.

```python
import numpy as np

data = [12, 15, 14, 13, 16, 11, 18]
bootstrap_means = [np.mean(np.random.choice(data, len(data))) for _ in range(10000)]
ci_lower = np.percentile(bootstrap_means, 2.5)
ci_upper = np.percentile(bootstrap_means, 97.5)
```

No assumptions about the underlying distribution. Works for any statistic (mean, median, P99, ratio).

---

## Key Takeaways

- Use median and percentiles for skewed data (response times, salaries). Mean hides tail behaviour.
- Confidence intervals quantify uncertainty; p-values quantify surprise. Neither proves anything absolutely.
- A/B testing requires pre-calculated sample sizes, fixed analysis points, and correction for multiple comparisons.
- Correlation ≠ causation. Controlled experiments (A/B tests) are the gold standard for causal inference.
- Linear regression is the foundation of prediction models. R² tells you how much variance is explained; RMSE tells you how wrong you are on average.
- The bootstrap is the most practical tool for uncertainty estimation — it requires no distributional assumptions.
