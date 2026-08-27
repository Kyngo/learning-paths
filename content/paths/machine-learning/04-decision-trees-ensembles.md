---
title: "Decision Trees & Ensembles"
weight: 4
---

# Decision Trees & Ensembles

Decision trees are the foundation of the most powerful classical ML algorithms. A single tree is intuitive and interpretable but prone to overfitting. Ensemble methods — random forests and gradient boosting — combine many trees to achieve state-of-the-art performance on tabular data.

---

## Decision Trees

A decision tree recursively splits the feature space into regions, making predictions based on the majority class (classification) or mean value (regression) in each region.

### How Splitting Works

At each node, the algorithm selects the feature and threshold that best separates the data:

| Criterion | Task | Formula | Interpretation |
|-----------|------|---------|---------------|
| Gini Impurity | Classification | `1 - Σpᵢ²` | Probability of misclassifying a random sample |
| Entropy | Classification | `-Σpᵢ log₂(pᵢ)` | Information content of the distribution |
| MSE | Regression | `Σ(yᵢ - ȳ)²/n` | Variance of target values in the node |

The algorithm greedily chooses the split that maximises **information gain** (reduction in impurity).

```python
from sklearn.tree import DecisionTreeClassifier, export_text
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

tree = DecisionTreeClassifier(max_depth=3, random_state=42)
tree.fit(X_train, y_train)

print(f"Train accuracy: {tree.score(X_train, y_train):.3f}")
print(f"Test accuracy:  {tree.score(X_test, y_test):.3f}")
print(export_text(tree, feature_names=load_iris().feature_names))
```

### Controlling Complexity

An unpruned decision tree will keep splitting until every leaf is pure — this overfits severely. Control depth with these hyperparameters:

| Parameter | Effect | Typical Range |
|-----------|--------|--------------|
| `max_depth` | Maximum tree depth | 3–20 |
| `min_samples_split` | Minimum samples to split a node | 2–20 |
| `min_samples_leaf` | Minimum samples in a leaf | 1–10 |
| `max_features` | Number of features considered per split | sqrt(n), log2(n) |
| `max_leaf_nodes` | Maximum number of leaf nodes | 10–100 |

### Advantages and Limitations

| ✅ Advantages | ❌ Limitations |
|--------------|---------------|
| Highly interpretable | Prone to overfitting |
| No scaling needed | Unstable (small data changes → different tree) |
| Handles mixed types | Greedy splits (not globally optimal) |
| Fast training and inference | Poor extrapolation |

---

## Random Forests

A random forest is an ensemble of decision trees trained with two sources of randomness:

1. **Bagging (Bootstrap Aggregating)**: Each tree is trained on a random sample with replacement
2. **Feature randomness**: Each split considers only a random subset of features

The final prediction is the **majority vote** (classification) or **average** (regression) across all trees.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report

rf = RandomForestClassifier(
    n_estimators=100,      # Number of trees
    max_depth=10,
    max_features="sqrt",   # sqrt(n_features) per split
    min_samples_leaf=2,
    random_state=42,
    n_jobs=-1              # Parallelise across all CPU cores
)
rf.fit(X_train, y_train)

y_pred = rf.predict(X_test)
print(classification_report(y_test, y_pred))
```

### Why It Works

Each individual tree is a **weak learner** — high variance, potentially biased. By averaging many decorrelated trees, the variance component cancels out while bias remains low. The two sources of randomness ensure the trees are sufficiently different.

### Key Hyperparameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `n_estimators` | 100 | More trees → lower variance, higher compute |
| `max_depth` | None (unlimited) | Limits tree complexity |
| `max_features` | "sqrt" (clf) / 1.0 (reg) | Controls tree correlation |
| `min_samples_leaf` | 1 | Prevents tiny leaves |
| `bootstrap` | True | Whether to use bagging |
| `oob_score` | False | Out-of-bag evaluation (free validation) |

---

## Gradient Boosting

Gradient boosting builds trees **sequentially**, where each tree corrects the errors of the previous ensemble. Instead of averaging independent trees, it performs gradient descent in function space.

### How It Works

1. Fit a simple model (e.g., predict the mean)
2. Compute the **residuals** (errors) of the current prediction
3. Fit a new tree to predict those residuals
4. Add the new tree's predictions (scaled by learning rate) to the ensemble
5. Repeat

```python
from sklearn.ensemble import GradientBoostingClassifier

gb = GradientBoostingClassifier(
    n_estimators=200,
    learning_rate=0.1,
    max_depth=3,           # Boosting uses shallow trees
    subsample=0.8,         # Stochastic gradient boosting
    random_state=42
)
gb.fit(X_train, y_train)
print(f"Accuracy: {gb.score(X_test, y_test):.3f}")
```

### XGBoost

The most popular gradient boosting implementation. Adds regularisation, parallel tree construction, and efficient handling of sparse data.

```python
from xgboost import XGBClassifier

xgb = XGBClassifier(
    n_estimators=300,
    learning_rate=0.05,
    max_depth=6,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.1,         # L1 regularisation
    reg_lambda=1.0,        # L2 regularisation
    random_state=42,
    use_label_encoder=False,
    eval_metric="logloss"
)
xgb.fit(X_train, y_train)
print(f"Accuracy: {xgb.score(X_test, y_test):.3f}")
```

### LightGBM

Uses **leaf-wise** tree growth (vs level-wise in XGBoost) and histogram-based splitting for faster training on large datasets.

```python
from lightgbm import LGBMClassifier

lgbm = LGBMClassifier(
    n_estimators=300,
    learning_rate=0.05,
    num_leaves=31,         # Controls complexity (instead of max_depth)
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.1,
    reg_lambda=1.0,
    random_state=42,
    verbose=-1
)
lgbm.fit(X_train, y_train)
print(f"Accuracy: {lgbm.score(X_test, y_test):.3f}")
```

### CatBoost

Handles categorical features natively (no encoding needed) and uses **ordered boosting** to reduce prediction shift.

```python
from catboost import CatBoostClassifier

cb = CatBoostClassifier(
    iterations=300,
    learning_rate=0.05,
    depth=6,
    l2_leaf_reg=3.0,
    random_state=42,
    verbose=0
)
cb.fit(X_train, y_train)
print(f"Accuracy: {cb.score(X_test, y_test):.3f}")
```

### Comparison

| Library | Tree Growth | Categorical Support | Speed | Best For |
|---------|-----------|-------------------|-------|----------|
| sklearn GBM | Level-wise | No | Slowest | Learning, small datasets |
| XGBoost | Level-wise (default) | Limited (experimental) | Fast | Competitions, production |
| LightGBM | Leaf-wise | Yes (integer-encoded) | Fastest | Large datasets |
| CatBoost | Symmetric | Native (best) | Fast | Categorical-heavy data |

---

## Feature Importance

Tree-based models provide built-in feature importance scores.

### Impurity-Based Importance

How much each feature reduces impurity across all splits:

```python
import pandas as pd
import matplotlib.pyplot as plt

importances = rf.feature_importances_
feature_names = load_iris().feature_names

feat_imp = pd.Series(importances, index=feature_names).sort_values(ascending=True)
feat_imp.plot.barh()
plt.xlabel("Importance (Gini)")
plt.title("Feature Importance — Random Forest")
plt.tight_layout()
plt.show()
```

**Caveat:** Impurity-based importance is biased towards high-cardinality features and features with many unique values. Prefer **permutation importance** for more reliable results (Section 9).

---

## Bagging vs Boosting

| Aspect | Bagging (Random Forest) | Boosting (GBM) |
|--------|------------------------|----------------|
| Tree construction | Independent, parallel | Sequential, each corrects previous |
| Goal | Reduce variance | Reduce bias |
| Tree depth | Deep trees (low bias) | Shallow trees (high bias individually) |
| Overfitting risk | Lower | Higher (mitigated by learning rate) |
| Training speed | Parallelisable | Sequential |
| Typical use case | Robust baseline | Maximum accuracy on tabular data |

---

## Key Takeaways

- A single decision tree is interpretable but overfits easily — always constrain depth, leaf size, or number of leaf nodes.
- Random forests reduce variance by averaging many decorrelated trees — they are a strong, robust default for most tabular problems.
- Gradient boosting builds trees sequentially to correct errors, achieving higher accuracy than bagging at the cost of sequential training and overfitting risk.
- XGBoost, LightGBM, and CatBoost are the production-grade boosting libraries — LightGBM is fastest, CatBoost handles categoricals best, XGBoost is the most widely adopted.
- Feature importance from trees gives a quick signal but is biased — use permutation importance for reliable results.
- For most tabular datasets, gradient boosted trees are the strongest classical ML algorithm and the default choice before considering deep learning.
