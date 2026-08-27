---
title: "Introduction to Machine Learning"
weight: 1
---

# Introduction to Machine Learning

Machine learning is the study of algorithms that improve their performance on a task through experience (data), without being explicitly programmed for that task. This section establishes the taxonomy, core tradeoffs, and workflow that every subsequent section builds upon.

---

## What Is Machine Learning?

Traditional programming: you write rules, the program applies them to data and produces output.

Machine learning: you provide data and the desired output, and the algorithm learns the rules.

```text
Traditional:  Rules + Data  → Program → Output
ML:           Data + Output → Algorithm → Rules (Model)
```

A **model** is the learned function that maps inputs to outputs. **Training** is the process of finding that function. **Inference** is applying the trained model to new data.

---

## Types of Machine Learning

| Type | Input | Goal | Examples |
|------|-------|------|----------|
| Supervised | Labelled data (X, y) | Predict y from X | Spam detection, house pricing, medical diagnosis |
| Unsupervised | Unlabelled data (X only) | Discover structure | Customer segmentation, anomaly detection, topic modelling |
| Reinforcement | Environment + rewards | Maximise cumulative reward | Game playing, robotics, ad bidding |
| Semi-supervised | Small labelled + large unlabelled | Leverage both | Text classification with few labels |
| Self-supervised | Unlabelled (creates own labels) | Learn representations | Word embeddings, image pretraining |

### Supervised Learning

The most common paradigm. You have input features **X** and target labels **y**. The model learns a function `f(X) ≈ y`.

Two sub-types:

- **Classification** — y is categorical (spam/not-spam, digit 0–9, disease type)
- **Regression** — y is continuous (price, temperature, probability)

```python
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = LogisticRegression(max_iter=200)
model.fit(X_train, y_train)
print(f"Accuracy: {model.score(X_test, y_test):.2f}")
```

### Unsupervised Learning

No labels. The algorithm finds patterns, groupings, or compressed representations in the data.

- **Clustering** — group similar data points (K-Means, DBSCAN)
- **Dimensionality reduction** — compress features while preserving structure (PCA, t-SNE)
- **Anomaly detection** — identify data points that do not fit the normal pattern

### Reinforcement Learning

An agent interacts with an environment, takes actions, and receives rewards. It learns a policy that maximises cumulative reward over time. Not covered in depth in this classical ML path — the focus here is supervised and unsupervised methods.

---

## The ML Workflow

Every ML project follows this lifecycle, regardless of algorithm:

```text
1. Define the problem     → What are you predicting? What metric matters?
2. Collect data           → Sources, volume, quality assessment
3. Explore & clean data   → EDA, missing values, outliers, distributions
4. Feature engineering    → Transform raw data into model-ready features
5. Select & train model   → Choose algorithm, fit on training data
6. Evaluate               → Measure performance on held-out data
7. Tune hyperparameters   → Optimise model configuration
8. Deploy & monitor       → Serve predictions, detect drift
```

Steps 3–7 are iterative. You rarely get them right on the first pass.

### Problem Framing

The single most impactful decision is how you frame the problem:

| Business Question | ML Framing | Type |
|-------------------|-----------|------|
| Will this customer churn? | Binary classification (churn/stay) | Supervised |
| What price should we set? | Regression (continuous price) | Supervised |
| Which customers are similar? | Clustering | Unsupervised |
| Is this transaction fraudulent? | Binary classification or anomaly detection | Both possible |
| How many units will we sell next month? | Regression / time series forecasting | Supervised |

---

## Bias-Variance Tradeoff

The most fundamental concept in ML. Every model's prediction error can be decomposed:

```text
Total Error = Bias² + Variance + Irreducible Noise
```

| Component | Meaning | Symptom |
|-----------|---------|---------|
| **Bias** | Error from oversimplified assumptions | Model misses patterns (underfitting) |
| **Variance** | Error from sensitivity to training data fluctuations | Model memorises noise (overfitting) |
| **Irreducible noise** | Inherent randomness in the data | Cannot be reduced by any model |

### Underfitting (High Bias)

The model is too simple to capture the underlying pattern.

- Training error: **high**
- Validation error: **high**
- Gap between them: **small**

Fixes: use a more complex model, add features, reduce regularisation.

### Overfitting (High Variance)

The model memorises training data, including its noise.

- Training error: **low**
- Validation error: **high**
- Gap between them: **large**

Fixes: more training data, regularisation, simpler model, feature selection, ensemble methods.

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.preprocessing import PolynomialFeatures
from sklearn.linear_model import LinearRegression
from sklearn.pipeline import make_pipeline

# Generate noisy data
np.random.seed(42)
X = np.sort(np.random.uniform(0, 1, 30)).reshape(-1, 1)
y = np.sin(2 * np.pi * X).ravel() + np.random.normal(0, 0.2, 30)

fig, axes = plt.subplots(1, 3, figsize=(15, 4))
X_plot = np.linspace(0, 1, 100).reshape(-1, 1)

for ax, degree, label in zip(axes, [1, 4, 15], ["Underfit", "Good fit", "Overfit"]):
    model = make_pipeline(PolynomialFeatures(degree), LinearRegression())
    model.fit(X, y)
    ax.scatter(X, y, color="black", s=20)
    ax.plot(X_plot, model.predict(X_plot), color="red")
    ax.set_title(f"{label} (degree={degree})")
    ax.set_ylim(-1.5, 1.5)

plt.tight_layout()
plt.show()
```

---

## Train / Validation / Test Splits

You need separate data for training, tuning, and final evaluation:

| Split | Purpose | Typical Size |
|-------|---------|-------------|
| **Training** | Fit the model parameters | 60–80% |
| **Validation** | Tune hyperparameters, select model | 10–20% |
| **Test** | Final, unbiased performance estimate | 10–20% |

**Critical rule:** The test set must never influence model development. If you peek at test results and adjust your model, you have contaminated the estimate.

```python
from sklearn.model_selection import train_test_split

# Two-stage split: train+val / test, then train / val
X_trainval, X_test, y_trainval, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)
X_train, X_val, y_train, y_val = train_test_split(
    X_trainval, y_trainval, test_size=0.25, random_state=42  # 0.25 × 0.8 = 0.2
)
```

In practice, **cross-validation** (Section 7) is preferred over a single validation split because it gives a more robust estimate from limited data.

---

## Parametric vs Non-Parametric Models

| Property | Parametric | Non-Parametric |
|----------|-----------|----------------|
| Assumption | Fixed functional form (e.g. linear) | No fixed form — complexity grows with data |
| Parameters | Fixed number, learned from data | Number grows with dataset size |
| Speed | Fast training and inference | Can be slow on large datasets |
| Data needs | Works with less data | Needs more data to avoid overfitting |
| Examples | Linear regression, logistic regression, naive Bayes | K-NN, decision trees, SVM (kernel) |

---

## No Free Lunch Theorem

There is no single algorithm that is best for every problem. The No Free Lunch theorem (Wolpert, 1996) states that averaged over all possible problems, no algorithm outperforms any other.

In practice, this means:

- **Understand your data** before choosing an algorithm
- **Try multiple approaches** and compare with proper evaluation
- **Domain knowledge** is your strongest lever — it guides feature engineering and model selection
- **Gradient boosted trees** (XGBoost, LightGBM) win most tabular data competitions, but that does not mean they are always the right choice in production

---

## Scikit-learn: The API You Will Use Everywhere

Scikit-learn provides a consistent API across all algorithms:

```python
from sklearn.ensemble import RandomForestClassifier

# 1. Instantiate with hyperparameters
model = RandomForestClassifier(n_estimators=100, max_depth=10, random_state=42)

# 2. Fit on training data
model.fit(X_train, y_train)

# 3. Predict on new data
predictions = model.predict(X_test)

# 4. Evaluate
accuracy = model.score(X_test, y_test)
```

Every estimator follows the same pattern: `__init__` → `fit()` → `predict()` (or `transform()` for preprocessors). This uniformity is what makes scikit-learn powerful — you can swap algorithms without changing your pipeline structure.

---

## Key Takeaways

- Machine learning builds models from data instead of explicit rules — supervised learning (labelled data) is the most common paradigm, unsupervised learning discovers structure without labels.
- The bias-variance tradeoff is the central tension: too simple underfits, too complex overfits. Every modelling decision navigates this tradeoff.
- Always split data into train/validation/test sets — never evaluate on data the model has seen during training or tuning.
- The ML workflow is iterative: define the problem, prepare data, train, evaluate, tune, deploy, monitor.
- There is no universally best algorithm — the right choice depends on your data, constraints, and domain.
- Scikit-learn's consistent `fit`/`predict`/`transform` API is the foundation for everything in this path.
