---
title: "Model Evaluation"
weight: 7
---

# Model Evaluation

A model is only as trustworthy as the method used to evaluate it. This section covers the metrics, validation strategies, and diagnostic tools that tell you whether your model genuinely works — or just memorised the training data.

---

## Classification Metrics

### Confusion Matrix

The confusion matrix is the foundation of all classification metrics:

```text
                  Predicted
                  Pos    Neg
Actual  Pos  [   TP  |  FN  ]
        Neg  [   FP  |  TN  ]
```

| Term | Meaning |
|------|---------|
| TP (True Positive) | Correctly predicted positive |
| TN (True Negative) | Correctly predicted negative |
| FP (False Positive) | Incorrectly predicted positive (Type I error) |
| FN (False Negative) | Incorrectly predicted negative (Type II error) |

```python
from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay
from sklearn.datasets import load_breast_cancer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = LogisticRegression(max_iter=5000, random_state=42)
model.fit(X_train, y_train)
y_pred = model.predict(X_test)

cm = confusion_matrix(y_test, y_pred)
ConfusionMatrixDisplay(cm, display_labels=["Malignant", "Benign"]).plot(cmap="Blues")
```

### Core Metrics

| Metric | Formula | When to Use |
|--------|---------|-------------|
| **Accuracy** | (TP+TN) / (TP+TN+FP+FN) | Balanced classes only |
| **Precision** | TP / (TP+FP) | Cost of false positives is high (spam filter) |
| **Recall (Sensitivity)** | TP / (TP+FN) | Cost of false negatives is high (disease detection) |
| **F1 Score** | 2 × (Precision × Recall) / (Precision + Recall) | Need balance between precision and recall |
| **Specificity** | TN / (TN+FP) | True negative rate matters |

```python
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

print(f"Accuracy:  {accuracy_score(y_test, y_pred):.4f}")
print(f"Precision: {precision_score(y_test, y_pred):.4f}")
print(f"Recall:    {recall_score(y_test, y_pred):.4f}")
print(f"F1:        {f1_score(y_test, y_pred):.4f}")
```

### Why Accuracy Fails on Imbalanced Data

If 95% of emails are not spam, a model that always predicts "not spam" achieves 95% accuracy while catching zero spam. Accuracy is misleading when classes are imbalanced.

### Classification Report

```python
from sklearn.metrics import classification_report

print(classification_report(y_test, y_pred, target_names=["Malignant", "Benign"]))
```

Output includes per-class precision, recall, F1, and the macro/weighted averages.

---

## ROC Curve and AUC

The **Receiver Operating Characteristic** curve plots TPR (recall) against FPR (1 - specificity) at every decision threshold.

```python
from sklearn.metrics import roc_curve, roc_auc_score
import matplotlib.pyplot as plt

y_proba = model.predict_proba(X_test)[:, 1]

fpr, tpr, thresholds = roc_curve(y_test, y_proba)
auc = roc_auc_score(y_test, y_proba)

plt.plot(fpr, tpr, label=f"AUC = {auc:.3f}")
plt.plot([0, 1], [0, 1], "k--", label="Random (AUC = 0.5)")
plt.xlabel("False Positive Rate")
plt.ylabel("True Positive Rate")
plt.title("ROC Curve")
plt.legend()
plt.show()
```

| AUC Value | Interpretation |
|-----------|---------------|
| 1.0 | Perfect classifier |
| 0.9–1.0 | Excellent |
| 0.8–0.9 | Good |
| 0.7–0.8 | Fair |
| 0.5 | Random guessing |
| < 0.5 | Worse than random (labels likely swapped) |

### Precision-Recall Curve

For imbalanced datasets, the PR curve is more informative than ROC:

```python
from sklearn.metrics import precision_recall_curve, average_precision_score

precision, recall, thresholds = precision_recall_curve(y_test, y_proba)
ap = average_precision_score(y_test, y_proba)

plt.plot(recall, precision, label=f"AP = {ap:.3f}")
plt.xlabel("Recall")
plt.ylabel("Precision")
plt.title("Precision-Recall Curve")
plt.legend()
plt.show()
```

---

## Regression Metrics

| Metric | Formula | Interpretation |
|--------|---------|---------------|
| **MSE** | Σ(y - ŷ)² / n | Average squared error (penalises large errors) |
| **RMSE** | √MSE | Same units as target |
| **MAE** | Σ\|y - ŷ\| / n | Average absolute error (robust to outliers) |
| **R²** | 1 - SS_res / SS_tot | Proportion of variance explained (1 = perfect) |
| **MAPE** | Σ(\|y - ŷ\| / \|y\|) / n × 100 | Percentage error (undefined when y = 0) |

```python
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
import numpy as np

# Example: regression evaluation
mse = mean_squared_error(y_test, y_pred)
rmse = np.sqrt(mse)
mae = mean_absolute_error(y_test, y_pred)
r2 = r2_score(y_test, y_pred)

print(f"MSE:  {mse:.2f}")
print(f"RMSE: {rmse:.2f}")
print(f"MAE:  {mae:.2f}")
print(f"R²:   {r2:.4f}")
```

### Choosing Between MSE and MAE

- **MSE/RMSE** — penalises large errors quadratically. Use when large errors are especially costly.
- **MAE** — treats all errors equally. More robust to outliers. Use when outliers in the target are expected but not catastrophic.

---

## Cross-Validation

A single train/test split is unreliable — the score depends on which specific data ends up in each set. Cross-validation gives a more robust estimate.

### K-Fold Cross-Validation

1. Split data into K equal folds
2. For each fold: train on K-1 folds, evaluate on the held-out fold
3. Report the mean and standard deviation of scores across all folds

```python
from sklearn.model_selection import cross_val_score

scores = cross_val_score(model, X, y, cv=5, scoring="accuracy")
print(f"CV Accuracy: {scores.mean():.4f} ± {scores.std():.4f}")
```

### Stratified K-Fold

For classification, preserves the class distribution in each fold:

```python
from sklearn.model_selection import StratifiedKFold

skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=skf, scoring="f1")
print(f"CV F1: {scores.mean():.4f} ± {scores.std():.4f}")
```

### Other CV Strategies

| Strategy | Use When |
|----------|----------|
| `KFold` | General-purpose |
| `StratifiedKFold` | Imbalanced classification |
| `GroupKFold` | Groups (patients, users) must not span folds |
| `TimeSeriesSplit` | Temporal data (no future leakage) |
| `RepeatedKFold` | Want more stable estimates (repeat K-fold N times) |
| `LeaveOneOut` | Very small datasets (K = n) |

```python
from sklearn.model_selection import GroupKFold, TimeSeriesSplit

# Group K-Fold: ensure same patient is not in both train and test
gkf = GroupKFold(n_splits=5)
# Pass groups= argument to cross_val_score
# scores = cross_val_score(model, X, y, cv=gkf, groups=patient_ids, scoring="f1")

# Time series split: always train on past, test on future
tscv = TimeSeriesSplit(n_splits=5)
# scores = cross_val_score(model, X, y, cv=tscv, scoring="neg_mean_squared_error")
```

---

## Learning Curves

Learning curves show how model performance changes with the amount of training data. They diagnose underfitting and overfitting.

```python
from sklearn.model_selection import learning_curve

train_sizes, train_scores, val_scores = learning_curve(
    model, X, y, cv=5,
    train_sizes=np.linspace(0.1, 1.0, 10),
    scoring="accuracy",
    n_jobs=-1
)

train_mean = train_scores.mean(axis=1)
train_std = train_scores.std(axis=1)
val_mean = val_scores.mean(axis=1)
val_std = val_scores.std(axis=1)

plt.fill_between(train_sizes, train_mean - train_std, train_mean + train_std, alpha=0.1, color="blue")
plt.fill_between(train_sizes, val_mean - val_std, val_mean + val_std, alpha=0.1, color="orange")
plt.plot(train_sizes, train_mean, "o-", color="blue", label="Training score")
plt.plot(train_sizes, val_mean, "o-", color="orange", label="Validation score")
plt.xlabel("Training Set Size")
plt.ylabel("Accuracy")
plt.title("Learning Curve")
plt.legend()
plt.show()
```

### Interpreting Learning Curves

| Pattern | Diagnosis | Action |
|---------|-----------|--------|
| Both curves converge at a high score | Good fit | None needed |
| Both curves converge at a low score | Underfitting (high bias) | More complex model, more features |
| Large gap between curves | Overfitting (high variance) | More data, regularisation, simpler model |
| Validation curve still rising | More data would help | Collect more training data |

---

## Choosing the Right Metric

| Scenario | Recommended Metric | Why |
|----------|-------------------|-----|
| Balanced classification | Accuracy or F1 | Accuracy is interpretable, F1 balances precision/recall |
| Imbalanced classification | F1, PR-AUC, or recall at fixed precision | Accuracy is misleading |
| Ranking / scoring | ROC-AUC | Threshold-independent |
| Cost-sensitive classification | Custom cost matrix or precision at fixed recall | Directly models business cost |
| Regression (no outliers) | RMSE | Penalises large errors |
| Regression (with outliers) | MAE | Robust to extreme values |
| Business reporting | MAPE, R² | Interpretable to stakeholders |

---

## Key Takeaways

- Accuracy is only meaningful for balanced classes — use precision, recall, F1, or AUC for imbalanced problems.
- The confusion matrix is the source of truth: every classification metric derives from TP, TN, FP, FN.
- ROC-AUC evaluates across all thresholds and is useful for ranking; PR-AUC is more informative for imbalanced datasets.
- Cross-validation (especially stratified K-fold) gives far more reliable estimates than a single train/test split — always report mean ± standard deviation.
- Learning curves diagnose whether your model needs more data, more complexity, or more regularisation.
- Choose your evaluation metric before training — it should reflect the real-world cost of errors, not just statistical convenience.
