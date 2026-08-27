---
title: "Support Vector Machines"
weight: 5
---

# Support Vector Machines

Support Vector Machines (SVMs) find the optimal separating hyperplane between classes by maximising the margin — the distance between the decision boundary and the nearest data points. With the kernel trick, SVMs can learn non-linear boundaries while working in high-dimensional space efficiently.

---

## Linear SVM

### The Maximum Margin Classifier

Given two linearly separable classes, infinitely many hyperplanes can separate them. SVM chooses the one with the **largest margin** — the greatest distance to the nearest training points from either class.

```text
Hyperplane:  w · x + b = 0
Margin:      2 / ||w||
Objective:   Minimise ||w||² subject to yᵢ(w · xᵢ + b) ≥ 1 for all i
```

The training points that lie exactly on the margin boundary are called **support vectors**. They are the only points that influence the decision boundary — all other points can be removed without changing the model.

```python
from sklearn.svm import SVC
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
import numpy as np

X, y = make_classification(n_samples=200, n_features=2, n_redundant=0,
                           n_clusters_per_class=1, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# SVM requires scaled features
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

svm = SVC(kernel="linear", C=1.0)
svm.fit(X_train_scaled, y_train)

print(f"Accuracy: {svm.score(X_test_scaled, y_test):.3f}")
print(f"Support vectors: {svm.n_support_}")
```

### Visualising the Decision Boundary

```python
import matplotlib.pyplot as plt

def plot_svm_boundary(model, X, y, title="SVM Decision Boundary"):
    h = 0.02
    x_min, x_max = X[:, 0].min() - 1, X[:, 0].max() + 1
    y_min, y_max = X[:, 1].min() - 1, X[:, 1].max() + 1
    xx, yy = np.meshgrid(np.arange(x_min, x_max, h), np.arange(y_min, y_max, h))
    Z = model.predict(np.c_[xx.ravel(), yy.ravel()])
    Z = Z.reshape(xx.shape)

    plt.contourf(xx, yy, Z, alpha=0.3, cmap="RdBu")
    plt.scatter(X[:, 0], X[:, 1], c=y, cmap="RdBu", edgecolors="black", s=30)
    # Highlight support vectors
    sv = model.support_vectors_
    plt.scatter(sv[:, 0], sv[:, 1], s=100, facecolors="none", edgecolors="green", linewidths=2)
    plt.title(title)
    plt.show()

plot_svm_boundary(svm, X_train_scaled, y_train, "Linear SVM")
```

---

## Soft Margin and the C Parameter

Real data is rarely perfectly separable. The **soft margin** SVM allows some misclassifications by introducing **slack variables** ξᵢ:

```text
Minimise:  (1/2)||w||² + C × Σξᵢ
Subject to: yᵢ(w · xᵢ + b) ≥ 1 - ξᵢ,  ξᵢ ≥ 0
```

The `C` parameter controls the tradeoff:

| C Value | Effect | Behaviour |
|---------|--------|-----------|
| Large C | High penalty for misclassification | Narrow margin, fewer errors on training data, risk of overfitting |
| Small C | Low penalty for misclassification | Wide margin, more tolerant of errors, better generalisation |

```python
fig, axes = plt.subplots(1, 3, figsize=(15, 4))

for ax, C in zip(axes, [0.01, 1.0, 100.0]):
    model = SVC(kernel="linear", C=C)
    model.fit(X_train_scaled, y_train)
    ax.set_title(f"C={C} (SVs={model.n_support_.sum()})")

    h = 0.02
    xx, yy = np.meshgrid(
        np.arange(X_train_scaled[:, 0].min()-1, X_train_scaled[:, 0].max()+1, h),
        np.arange(X_train_scaled[:, 1].min()-1, X_train_scaled[:, 1].max()+1, h)
    )
    Z = model.predict(np.c_[xx.ravel(), yy.ravel()]).reshape(xx.shape)
    ax.contourf(xx, yy, Z, alpha=0.3, cmap="RdBu")
    ax.scatter(X_train_scaled[:, 0], X_train_scaled[:, 1], c=y_train,
               cmap="RdBu", edgecolors="black", s=20)

plt.tight_layout()
plt.show()
```

---

## The Kernel Trick

When data is not linearly separable, SVM can map it to a higher-dimensional space where a linear separator exists. The **kernel trick** computes the dot product in this high-dimensional space without explicitly performing the transformation.

### Common Kernels

| Kernel | Formula | Use When |
|--------|---------|----------|
| Linear | `K(x, x') = x · x'` | Linearly separable data, high-dimensional sparse data |
| RBF (Gaussian) | `K(x, x') = exp(-γ\|\|x-x'\|\|²)` | Default choice, works well in most cases |
| Polynomial | `K(x, x') = (γ(x · x') + r)^d` | Feature interactions of specific degree matter |
| Sigmoid | `K(x, x') = tanh(γ(x · x') + r)` | Rarely used (similar to neural networks) |

### RBF Kernel

The most commonly used kernel. The `gamma` parameter controls the influence radius of each training point:

- **Large gamma** → each point influences only nearby points (complex boundary, risk of overfitting)
- **Small gamma** → each point influences a wide area (smooth boundary, risk of underfitting)

```python
from sklearn.datasets import make_moons

X, y = make_moons(n_samples=300, noise=0.15, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

svm_rbf = SVC(kernel="rbf", C=1.0, gamma="scale")
svm_rbf.fit(X_train_scaled, y_train)
print(f"Accuracy: {svm_rbf.score(X_test_scaled, y_test):.3f}")
```

### Effect of Gamma

```python
fig, axes = plt.subplots(1, 3, figsize=(15, 4))

for ax, gamma in zip(axes, [0.1, 1.0, 10.0]):
    model = SVC(kernel="rbf", C=1.0, gamma=gamma)
    model.fit(X_train_scaled, y_train)
    test_acc = model.score(X_test_scaled, y_test)
    ax.set_title(f"gamma={gamma} (acc={test_acc:.2f})")

    h = 0.02
    xx, yy = np.meshgrid(
        np.arange(X_train_scaled[:, 0].min()-1, X_train_scaled[:, 0].max()+1, h),
        np.arange(X_train_scaled[:, 1].min()-1, X_train_scaled[:, 1].max()+1, h)
    )
    Z = model.predict(np.c_[xx.ravel(), yy.ravel()]).reshape(xx.shape)
    ax.contourf(xx, yy, Z, alpha=0.3, cmap="RdBu")
    ax.scatter(X_train_scaled[:, 0], X_train_scaled[:, 1], c=y_train,
               cmap="RdBu", edgecolors="black", s=20)

plt.tight_layout()
plt.show()
```

### Polynomial Kernel

Captures feature interactions up to a specified degree:

```python
svm_poly = SVC(kernel="poly", degree=3, C=1.0, gamma="scale", coef0=1)
svm_poly.fit(X_train_scaled, y_train)
print(f"Accuracy: {svm_poly.score(X_test_scaled, y_test):.3f}")
```

---

## Support Vector Regression (SVR)

SVM can also perform regression. Instead of maximising the margin between classes, SVR fits a tube of width `ε` around the data and penalises points outside it.

```python
from sklearn.svm import SVR
from sklearn.datasets import make_regression
from sklearn.metrics import mean_squared_error

X, y = make_regression(n_samples=200, n_features=5, noise=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

svr = SVR(kernel="rbf", C=100, gamma="scale", epsilon=0.1)
svr.fit(X_train_scaled, y_train)

y_pred = svr.predict(X_test_scaled)
print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.2f}")
```

### SVR Hyperparameters

| Parameter | Effect |
|-----------|--------|
| `C` | Penalty for points outside the ε-tube |
| `epsilon` | Width of the insensitive tube (points inside are not penalised) |
| `kernel` | Same options as SVC (linear, rbf, poly) |
| `gamma` | Kernel coefficient (for rbf, poly, sigmoid) |

---

## Practical Considerations

### When to Use SVM

| ✅ Good Fit | ❌ Poor Fit |
|------------|-----------|
| Small to medium datasets (< 50k samples) | Very large datasets (slow O(n²) to O(n³) training) |
| High-dimensional data (text, genomics) | Need probability estimates (SVM outputs distances, not probabilities) |
| Clear margin of separation exists | Need interpretable feature importance |
| Binary or small multiclass | Very noisy data with overlapping classes |

### Probability Estimates

SVM does not natively output probabilities. Scikit-learn uses Platt scaling (sigmoid calibration) when `probability=True`, but this is computationally expensive and adds a post-processing step:

```python
svm_proba = SVC(kernel="rbf", C=1.0, gamma="scale", probability=True)
svm_proba.fit(X_train_scaled, y_train)
probabilities = svm_proba.predict_proba(X_test_scaled)
```

### LinearSVC for Large Datasets

For linear kernels on large datasets, use `LinearSVC` — it uses liblinear instead of libsvm and scales much better:

```python
from sklearn.svm import LinearSVC

linear_svm = LinearSVC(C=1.0, max_iter=10000)
linear_svm.fit(X_train_scaled, y_train)
```

---

## Key Takeaways

- SVM finds the hyperplane that maximises the margin between classes — only the support vectors (points on the margin) determine the boundary.
- The C parameter controls the bias-variance tradeoff: large C prioritises correct classification (narrow margin), small C allows more errors (wide margin).
- The kernel trick maps data to higher dimensions without explicit transformation — RBF is the default choice for non-linear problems.
- Gamma controls RBF kernel influence radius: large gamma overfits (complex boundaries), small gamma underfits (smooth boundaries).
- SVM requires feature scaling (it is distance-based) and struggles with very large datasets — prefer `LinearSVC` for linear kernels at scale.
- For tabular data, gradient boosted trees typically outperform SVM with less tuning — SVM shines on high-dimensional data and small datasets.
