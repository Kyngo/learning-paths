---
title: "Linear Models"
weight: 3
---

# Linear Models

Linear models are the foundation of machine learning. They are fast, interpretable, and surprisingly effective. Every ML practitioner should understand them thoroughly — both because they serve as strong baselines and because the concepts (gradient descent, regularisation, loss functions) transfer to every other algorithm.

---

## Linear Regression

The simplest supervised model. It fits a linear function to predict a continuous target:

```text
ŷ = w₁x₁ + w₂x₂ + ... + wₙxₙ + b
```

Where **w** are the weights (coefficients), **b** is the bias (intercept), and **ŷ** is the prediction.

### Ordinary Least Squares (OLS)

OLS minimises the sum of squared residuals:

```text
Loss = Σ(yᵢ - ŷᵢ)² = Σ(yᵢ - (Xw + b))²
```

This has a closed-form solution: `w = (XᵀX)⁻¹Xᵀy`

```python
from sklearn.linear_model import LinearRegression
from sklearn.datasets import make_regression
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, r2_score
import numpy as np

X, y = make_regression(n_samples=200, n_features=5, noise=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = LinearRegression()
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
print(f"MSE:  {mean_squared_error(y_test, y_pred):.2f}")
print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.2f}")
print(f"R²:   {r2_score(y_test, y_pred):.4f}")

# Inspect coefficients
for name, coef in zip([f"x{i}" for i in range(5)], model.coef_):
    print(f"  {name}: {coef:.3f}")
print(f"  intercept: {model.intercept_:.3f}")
```

### Assumptions

Linear regression assumes:

| Assumption | Meaning | Diagnostic |
|-----------|---------|-----------|
| Linearity | Target is a linear function of features | Residual plot (should show no pattern) |
| Independence | Observations are independent | Check for time/spatial correlation |
| Homoscedasticity | Constant variance of residuals | Residual vs predicted plot |
| Normality of residuals | Residuals are normally distributed | Q-Q plot |
| No multicollinearity | Features are not highly correlated | VIF (Variance Inflation Factor) |

Violations do not make the model useless, but they affect confidence intervals and hypothesis tests.

---

## Gradient Descent

For large datasets, the closed-form OLS solution is computationally expensive (matrix inversion is O(n³)). Gradient descent is an iterative optimisation algorithm that scales better.

### How It Works

1. Initialise weights randomly
2. Compute the loss (cost function)
3. Compute the gradient (partial derivatives of loss w.r.t. each weight)
4. Update weights: `w = w - α × ∇L`  where α is the learning rate
5. Repeat until convergence

### Variants

| Variant | Batch Size | Pros | Cons |
|---------|-----------|------|------|
| Batch GD | Full dataset | Stable convergence | Slow on large datasets |
| Stochastic GD (SGD) | 1 sample | Fast, can escape local minima | Noisy updates |
| Mini-batch GD | Small batch (32–256) | Best of both worlds | Requires batch size tuning |

```python
from sklearn.linear_model import SGDRegressor
from sklearn.preprocessing import StandardScaler

# SGD requires scaled features
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

sgd_model = SGDRegressor(
    loss="squared_error",
    learning_rate="invscaling",
    eta0=0.01,
    max_iter=1000,
    random_state=42
)
sgd_model.fit(X_train_scaled, y_train)
print(f"R²: {sgd_model.score(X_test_scaled, y_test):.4f}")
```

### Learning Rate

The learning rate α controls step size:

- **Too large** → overshoots the minimum, diverges
- **Too small** → converges very slowly
- **Just right** → smooth, efficient convergence

---

## Logistic Regression

Despite the name, logistic regression is a **classification** algorithm. It models the probability that an observation belongs to a class.

### The Sigmoid Function

Logistic regression applies the sigmoid function to the linear combination:

```text
P(y=1|X) = σ(Xw + b) = 1 / (1 + e^(-(Xw + b)))
```

The sigmoid maps any real number to (0, 1), producing a probability.

### Binary Classification

```python
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_breast_cancer
from sklearn.metrics import classification_report

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = LogisticRegression(max_iter=5000, random_state=42)
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
y_proba = model.predict_proba(X_test)[:, 1]  # Probability of positive class

print(classification_report(y_test, y_pred))
```

### Multiclass Classification

Logistic regression handles multiple classes using:

- **One-vs-Rest (OvR)**: Train one binary classifier per class
- **Multinomial**: Softmax function across all classes (single model)

```python
from sklearn.datasets import load_iris

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = LogisticRegression(multi_class="multinomial", max_iter=200, random_state=42)
model.fit(X_train, y_train)
print(f"Accuracy: {model.score(X_test, y_test):.2f}")
```

### Decision Boundary

The default threshold is 0.5: predict class 1 if P(y=1) ≥ 0.5. You can adjust this to trade precision for recall:

```python
# Lower threshold → more positive predictions (higher recall, lower precision)
threshold = 0.3
y_pred_custom = (y_proba >= threshold).astype(int)
```

---

## Regularisation

Regularisation adds a penalty term to the loss function to prevent overfitting by discouraging large weights.

### L1 Regularisation (Lasso)

Adds the absolute value of weights to the loss:

```text
Loss = Σ(yᵢ - ŷᵢ)² + α × Σ|wⱼ|
```

L1 drives some weights exactly to zero, performing **feature selection**.

```python
from sklearn.linear_model import Lasso

model = Lasso(alpha=0.1)
model.fit(X_train, y_train)
print(f"Non-zero coefficients: {np.sum(model.coef_ != 0)} / {len(model.coef_)}")
```

### L2 Regularisation (Ridge)

Adds the squared magnitude of weights:

```text
Loss = Σ(yᵢ - ŷᵢ)² + α × Σwⱼ²
```

L2 shrinks weights towards zero but rarely eliminates them entirely.

```python
from sklearn.linear_model import Ridge

model = Ridge(alpha=1.0)
model.fit(X_train, y_train)
print(f"R²: {model.score(X_test, y_test):.4f}")
```

### ElasticNet (L1 + L2)

Combines both penalties:

```text
Loss = Σ(yᵢ - ŷᵢ)² + α × (ρ × Σ|wⱼ| + (1-ρ)/2 × Σwⱼ²)
```

Where `ρ` (l1_ratio) controls the mix.

```python
from sklearn.linear_model import ElasticNet

model = ElasticNet(alpha=0.1, l1_ratio=0.5)  # 50% L1, 50% L2
model.fit(X_train, y_train)
```

### Comparison

| Regularisation | Penalty | Feature Selection | When to Use |
|---------------|---------|-------------------|-------------|
| None (OLS) | — | No | Few features, no collinearity |
| L1 (Lasso) | Σ\|wⱼ\| | Yes (sparse solutions) | Many features, want automatic selection |
| L2 (Ridge) | Σwⱼ² | No (shrinks all) | Multicollinearity, all features relevant |
| ElasticNet | Both | Partial | Groups of correlated features |

For logistic regression, regularisation is controlled by the `C` parameter (inverse of α): smaller `C` means stronger regularisation.

```python
# Logistic regression with L1 regularisation
model = LogisticRegression(penalty="l1", C=0.1, solver="saga", max_iter=5000)
```

---

## Polynomial Features

Linear models can learn non-linear relationships by creating polynomial and interaction features:

```python
from sklearn.preprocessing import PolynomialFeatures
from sklearn.pipeline import make_pipeline

# Degree 2: adds x², xy, y² etc.
model = make_pipeline(
    PolynomialFeatures(degree=2, include_bias=False),
    Ridge(alpha=1.0)
)
model.fit(X_train, y_train)
print(f"R²: {model.score(X_test, y_test):.4f}")
```

**Warning:** Polynomial features grow combinatorially. With `d` features and degree `p`, you get `C(d+p, p)` features. Degree 3 with 20 features produces 1,771 features. Always pair polynomial features with regularisation.

---

## Key Takeaways

- Linear regression minimises squared error and has a closed-form solution (OLS) — use it as your first baseline for any regression problem.
- Gradient descent is the iterative alternative that scales to large datasets — learning rate is the critical hyperparameter.
- Logistic regression is a classification algorithm that outputs probabilities via the sigmoid function — adjust the decision threshold to trade precision for recall.
- Regularisation prevents overfitting: L1 (Lasso) performs feature selection, L2 (Ridge) handles multicollinearity, ElasticNet combines both.
- Polynomial features let linear models capture non-linear relationships, but feature count grows combinatorially — always pair with regularisation.
- Always scale features before fitting linear models with regularisation, otherwise the penalty is applied unevenly across features of different scales.
