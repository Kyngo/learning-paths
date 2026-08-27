---
title: "Calculus Essentials"
weight: 10
---

# Calculus Essentials

Calculus is the mathematics of change and accumulation. This section covers the minimum viable calculus for engineers — enough to understand gradient descent, optimisation, and the mathematical foundations of machine learning without a full analysis course.

---

## Limits

A **limit** describes the value a function approaches as its input approaches some point.

```
lim (x → a) f(x) = L
```

Means: as x gets arbitrarily close to a, f(x) gets arbitrarily close to L.

### Key Limits

| Limit | Value | Why It Matters |
|-------|-------|----------------|
| lim (x→0) sin(x)/x | 1 | Foundation of trigonometric calculus |
| lim (x→∞) (1 + 1/x)ˣ | e ≈ 2.718 | Definition of e, compound interest |
| lim (x→∞) log(x)/x | 0 | Log grows slower than linear |
| lim (x→∞) xⁿ/eˣ | 0 | Exponential dominates any polynomial |
| lim (x→0⁺) x ln(x) | 0 | Used in entropy calculations |

### Continuity

A function is **continuous** at a point if the limit equals the function value:

```
lim (x → a) f(x) = f(a)
```

Discontinuities matter in computing: step functions (ReLU at 0), integer rounding, and hash bucket boundaries are all discontinuous.

---

## Derivatives

The **derivative** of a function measures its instantaneous rate of change:

```
f'(x) = lim (h → 0) [f(x + h) - f(x)] / h
```

Geometrically: the slope of the tangent line at point x.

### Derivative Rules

| Rule | Formula | Example |
|------|---------|---------|
| Constant | d/dx [c] = 0 | d/dx [5] = 0 |
| Power | d/dx [xⁿ] = nxⁿ⁻¹ | d/dx [x³] = 3x² |
| Sum | d/dx [f + g] = f' + g' | d/dx [x² + 3x] = 2x + 3 |
| Product | d/dx [fg] = f'g + fg' | d/dx [x · eˣ] = eˣ + xeˣ |
| Chain | d/dx [f(g(x))] = f'(g(x)) · g'(x) | d/dx [sin(x²)] = cos(x²) · 2x |
| Exponential | d/dx [eˣ] = eˣ | eˣ is its own derivative |
| Logarithm | d/dx [ln(x)] = 1/x | |

### Common Derivatives

| Function | Derivative |
|----------|-----------|
| xⁿ | nxⁿ⁻¹ |
| eˣ | eˣ |
| aˣ | aˣ ln(a) |
| ln(x) | 1/x |
| log_a(x) | 1/(x ln(a)) |
| sin(x) | cos(x) |
| cos(x) | -sin(x) |
| 1/x | -1/x² |
| √x | 1/(2√x) |

### The Chain Rule — The Most Important Rule

The chain rule computes derivatives of composed functions:

```
d/dx [f(g(x))] = f'(g(x)) · g'(x)
```

This is exactly what **backpropagation** in neural networks computes. A neural network is a composition of functions:

```
output = f₃(f₂(f₁(input)))
```

To compute how the output changes with respect to the input (or any weight), the chain rule decomposes it into a product of local derivatives — one per layer.

---

## Optimisation

### Finding Minima and Maxima

At a local minimum or maximum, the derivative is zero:

```
f'(x) = 0  →  x is a critical point
```

To classify:
- f''(x) > 0 → local minimum (concave up)
- f''(x) < 0 → local maximum (concave down)
- f''(x) = 0 → inconclusive (test further)

### Gradient Descent

The core optimisation algorithm in machine learning. Given a loss function L(θ), update parameters to reduce the loss:

```
θ_new = θ_old - α × ∇L(θ_old)
```

Where:
- α is the **learning rate** (step size)
- ∇L is the **gradient** (vector of partial derivatives)

**Intuition:** The gradient points "uphill." Subtracting it moves you "downhill" toward a minimum.

```
Loss landscape (1D):

      ╱╲
     ╱  ╲        ╱╲
    ╱    ╲      ╱  ╲
   ╱      ╲    ╱    ╲
  ╱        ╲  ╱      ╲
 ╱          ╲╱        ╲
             ← gradient descent moves toward valley
```

### Learning Rate Effects

| Learning Rate | Behaviour |
|--------------|-----------|
| Too small | Converges slowly, may get stuck in local minimum |
| Just right | Converges efficiently to a good minimum |
| Too large | Overshoots, oscillates, or diverges |

### Variants

| Variant | Modification |
|---------|-------------|
| SGD (Stochastic) | Use a random subset of data per step |
| Momentum | Add fraction of previous update to smooth oscillations |
| Adam | Adaptive learning rate per parameter (most popular) |
| Learning rate scheduling | Decrease α over time |

---

## Partial Derivatives

When a function has multiple variables, the **partial derivative** measures the rate of change with respect to one variable while holding others constant.

```
f(x, y) = x² + 3xy + y²

∂f/∂x = 2x + 3y    (treat y as constant)
∂f/∂y = 3x + 2y    (treat x as constant)
```

### The Gradient

The gradient is the vector of all partial derivatives:

```
∇f = [∂f/∂x, ∂f/∂y]
```

For f(x, y) = x² + 3xy + y²:

```
∇f = [2x + 3y, 3x + 2y]
```

The gradient points in the direction of steepest ascent. Its magnitude tells you how steep the slope is.

### In Neural Networks

A neural network with millions of parameters θ = [θ₁, θ₂, ..., θₙ] has a loss function L(θ). The gradient:

```
∇L = [∂L/∂θ₁, ∂L/∂θ₂, ..., ∂L/∂θₙ]
```

Backpropagation computes this gradient efficiently using the chain rule, layer by layer, in O(n) time.

---

## Integrals

The **integral** is the reverse of differentiation — it computes accumulated area under a curve.

### Definite Integral

```
∫ₐᵇ f(x) dx = area under f(x) from a to b
```

### Fundamental Theorem of Calculus

If F'(x) = f(x), then:

```
∫ₐᵇ f(x) dx = F(b) - F(a)
```

The integral of a rate of change gives the total change.

### Common Integrals

| Function | Integral |
|----------|---------|
| xⁿ | xⁿ⁺¹/(n+1) + C (n ≠ -1) |
| 1/x | ln\|x\| + C |
| eˣ | eˣ + C |
| cos(x) | sin(x) + C |
| sin(x) | -cos(x) + C |

### Engineering Applications of Integration

| Application | What Is Integrated |
|-------------|-------------------|
| Total data transferred | Bandwidth × time |
| Work done | Force × distance |
| Probability | PDF integrated over range = CDF |
| Expected value | x × PDF integrated over all x |
| Total error (MSE) | Squared error integrated/summed |
| Area under ROC curve (AUC) | True positive rate vs. false positive rate |

### Numerical Integration

When analytical integration is impossible (most real functions), approximate numerically:

| Method | Idea | Error |
|--------|------|-------|
| Rectangle (Riemann) | Sum of rectangles | O(1/n) |
| Trapezoidal | Sum of trapezoids | O(1/n²) |
| Simpson's | Quadratic interpolation | O(1/n⁴) |
| Monte Carlo | Random sampling | O(1/√n) |

Monte Carlo integration scales well to high dimensions where grid methods fail (the "curse of dimensionality").

---

## Convexity

A function is **convex** if the line segment between any two points on the graph lies above the graph:

```
f(λx + (1-λ)y) ≤ λf(x) + (1-λ)f(y)   for 0 ≤ λ ≤ 1
```

### Why Convexity Matters

| Property | Implication |
|----------|------------|
| Any local minimum is the global minimum | Gradient descent will find the best solution |
| No saddle points or local minima traps | Optimisation is reliable |
| Unique solution (if strictly convex) | No ambiguity |

**Convex loss functions:** MSE (mean squared error), cross-entropy (for logistic regression), SVMs. These are well-behaved for optimisation.

**Non-convex loss functions:** Neural network losses. Gradient descent finds "good" local minima but not guaranteed global. In practice, the high-dimensional loss landscape has many equally good minima.

### Jensen's Inequality

For a convex function f:

```
f(E[X]) ≤ E[f(X)]
```

**Application:** This is why the log-likelihood is used instead of the likelihood in ML — it is numerically more stable and the inequality ensures optimisation still works.

---

## Taylor Series

Any smooth function can be approximated as a polynomial around a point a:

```
f(x) = f(a) + f'(a)(x-a) + f''(a)(x-a)²/2! + f'''(a)(x-a)³/3! + ...
```

### Common Taylor Series (around a = 0)

| Function | Series |
|----------|--------|
| eˣ | 1 + x + x²/2! + x³/3! + ... |
| sin(x) | x - x³/3! + x⁵/5! - ... |
| cos(x) | 1 - x²/2! + x⁴/4! - ... |
| ln(1+x) | x - x²/2 + x³/3 - ... (for \|x\| ≤ 1) |
| 1/(1-x) | 1 + x + x² + x³ + ... (for \|x\| < 1) |

### Why Taylor Series Matter

1. **Linearisation:** For small x, eˣ ≈ 1 + x. This approximation is used throughout physics and engineering.
2. **Numerical methods:** Functions are computed on CPUs using truncated Taylor series (CORDIC, minimax polynomials).
3. **Error analysis:** The remainder term tells you how accurate your approximation is.
4. **Softmax stability:** The softmax trick (subtract max) works because of the properties of the exponential Taylor series.

---

## Key Takeaways

- The derivative measures instantaneous rate of change. The chain rule decomposes derivatives of composed functions — this IS backpropagation.
- Gradient descent iteratively moves toward a minimum by following the negative gradient. The learning rate controls step size.
- Partial derivatives let you compute how a function changes with respect to each variable independently. The gradient collects all partial derivatives into a vector.
- Integrals compute accumulated quantities (area, probability, total work). Monte Carlo integration scales to high dimensions.
- Convex functions have no local minima traps — any local minimum is global. Neural network losses are non-convex but behave well in practice.
- You do not need to be fluent in calculus to use ML — but understanding the chain rule, gradient descent, and convexity gives you the intuition to debug training failures and choose appropriate architectures.
