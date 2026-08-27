---
title: "Feature Selection & Engineering"
weight: 9
---

# Feature Selection & Engineering

Features are the raw material of machine learning. Good features make simple models powerful; poor features make complex models weak. This section covers systematic methods for selecting the most informative features and engineering new ones from raw data.

---

## Why Feature Selection Matters

| Benefit | Explanation |
|---------|------------|
| Reduced overfitting | Fewer irrelevant features means less noise for the model to memorise |
| Faster training | Fewer features → smaller matrices → faster computation |
| Better interpretability | Easier to explain a model with 10 features than 500 |
| Improved accuracy | Removing noise features can improve generalisation |

The challenge: with `n` features, there are `2ⁿ` possible subsets. Exhaustive search is intractable for any non-trivial feature count.

---

## Filter Methods

Filter methods evaluate features **independently** of the model, using statistical measures. They are fast and model-agnostic.

### Variance Threshold

Remove features with near-zero variance (constant or near-constant):

```python
from sklearn.feature_selection import VarianceThreshold

selector = VarianceThreshold(threshold=0.01)
X_filtered = selector.fit_transform(X_train)
print(f"Features before: {X_train.shape[1]}, after: {X_filtered.shape[1]}")
```

### Correlation Filter

Remove one of two highly correlated features (they carry redundant information):

```python
import pandas as pd
import numpy as np

def drop_correlated(df, threshold=0.95):
    corr_matrix = df.corr().abs()
    upper = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
    to_drop = [col for col in upper.columns if any(upper[col] > threshold)]
    return df.drop(columns=to_drop), to_drop

X_clean, dropped = drop_correlated(pd.DataFrame(X_train), threshold=0.95)
print(f"Dropped {len(dropped)} highly correlated features")
```

### Mutual Information

Measures the dependency between a feature and the target. Works for both continuous and categorical features, captures non-linear relationships.

```python
from sklearn.feature_selection import mutual_info_classif, SelectKBest

# Select top K features by mutual information
selector = SelectKBest(score_func=mutual_info_classif, k=10)
X_selected = selector.fit_transform(X_train, y_train)

# Inspect scores
scores = pd.Series(selector.scores_, index=feature_names).sort_values(ascending=False)
print(scores.head(15))
```

### Statistical Tests

| Test | Feature Type | Target Type | scikit-learn Function |
|------|-------------|------------|----------------------|
| F-test (ANOVA) | Continuous | Categorical | `f_classif` |
| F-test (regression) | Continuous | Continuous | `f_regression` |
| Chi-squared | Categorical (non-negative) | Categorical | `chi2` |
| Mutual Information | Any | Any | `mutual_info_classif` / `mutual_info_regression` |

```python
from sklearn.feature_selection import f_classif, chi2

# ANOVA F-test for classification
selector = SelectKBest(f_classif, k=10)
X_selected = selector.fit_transform(X_train, y_train)
```

---

## Wrapper Methods

Wrapper methods evaluate feature subsets by training a model on each subset. More computationally expensive but capture feature interactions.

### Recursive Feature Elimination (RFE)

Iteratively removes the least important features:

```python
from sklearn.feature_selection import RFE
from sklearn.ensemble import RandomForestClassifier

model = RandomForestClassifier(n_estimators=100, random_state=42)
rfe = RFE(estimator=model, n_features_to_select=10, step=1)
rfe.fit(X_train, y_train)

selected = [name for name, selected in zip(feature_names, rfe.support_) if selected]
print(f"Selected features: {selected}")
print(f"Feature ranking: {dict(zip(feature_names, rfe.ranking_))}")
```

### RFECV (with Cross-Validation)

Automatically determines the optimal number of features:

```python
from sklearn.feature_selection import RFECV

rfecv = RFECV(
    estimator=RandomForestClassifier(n_estimators=100, random_state=42),
    step=1,
    cv=5,
    scoring="f1",
    n_jobs=-1
)
rfecv.fit(X_train, y_train)

print(f"Optimal features: {rfecv.n_features_}")
print(f"CV Score: {rfecv.cv_results_['mean_test_score'].max():.4f}")
```

---

## Embedded Methods

Embedded methods perform feature selection as part of the model training process.

### L1 Regularisation (Lasso)

L1 penalty drives coefficients to exactly zero, effectively selecting features:

```python
from sklearn.linear_model import LogisticRegression
from sklearn.feature_selection import SelectFromModel

# L1 logistic regression
l1_model = LogisticRegression(penalty="l1", C=0.1, solver="saga", max_iter=5000)
l1_model.fit(X_train, y_train)

selector = SelectFromModel(l1_model, prefit=True)
X_selected = selector.transform(X_train)
print(f"Non-zero features: {X_selected.shape[1]} / {X_train.shape[1]}")
```

### Tree-Based Feature Importance

```python
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(n_estimators=200, random_state=42)
rf.fit(X_train, y_train)

importances = pd.Series(rf.feature_importances_, index=feature_names)
top_features = importances.nlargest(15)
top_features.plot.barh()
```

---

## Permutation Importance

A model-agnostic method that measures how much performance drops when a feature's values are shuffled randomly. More reliable than impurity-based importance.

```python
from sklearn.inspection import permutation_importance

result = permutation_importance(
    rf, X_test, y_test,
    n_repeats=10,
    random_state=42,
    n_jobs=-1,
    scoring="f1"
)

perm_imp = pd.Series(result.importances_mean, index=feature_names).sort_values(ascending=False)
print(perm_imp.head(10))

# Plot with error bars
fig, ax = plt.subplots(figsize=(8, 6))
top_n = perm_imp.head(15)
ax.barh(range(len(top_n)), top_n.values)
ax.set_yticks(range(len(top_n)))
ax.set_yticklabels(top_n.index)
ax.set_xlabel("Mean Accuracy Decrease")
ax.set_title("Permutation Importance")
plt.tight_layout()
plt.show()
```

### Impurity vs Permutation Importance

| Aspect | Impurity-Based | Permutation |
|--------|---------------|-------------|
| Speed | Fast (computed during training) | Slower (requires multiple predictions) |
| Bias | Biased towards high-cardinality features | Unbiased |
| Requires test data | No | Yes (use validation set) |
| Captures interactions | Partially | Yes (measures total impact) |
| Model type | Tree-based only | Any model |

---

## Feature Engineering Techniques

### Interaction Features

Create new features by combining existing ones:

```python
import pandas as pd

df["rooms_per_person"] = df["total_rooms"] / df["population"]
df["bedrooms_ratio"] = df["total_bedrooms"] / df["total_rooms"]
df["income_per_room"] = df["median_income"] / df["total_rooms"]
```

### Binning

Convert continuous features into categories:

```python
# Equal-width bins
df["age_bin"] = pd.cut(df["age"], bins=[0, 18, 35, 55, 100], labels=["young", "adult", "middle", "senior"])

# Quantile-based bins (equal-frequency)
df["income_quartile"] = pd.qcut(df["income"], q=4, labels=["Q1", "Q2", "Q3", "Q4"])
```

### Target Encoding

Replace a categorical feature with the mean of the target for that category. Powerful but risky — must be computed within cross-validation to avoid leakage.

```python
from sklearn.model_selection import KFold
import numpy as np

def target_encode_cv(df, col, target, n_splits=5):
    encoded = pd.Series(np.nan, index=df.index)
    kf = KFold(n_splits=n_splits, shuffle=True, random_state=42)

    for train_idx, val_idx in kf.split(df):
        means = df.iloc[train_idx].groupby(col)[target].mean()
        encoded.iloc[val_idx] = df.iloc[val_idx][col].map(means)

    # Fill any NaN with global mean
    encoded.fillna(df[target].mean(), inplace=True)
    return encoded

df["city_target_enc"] = target_encode_cv(df, "city", "price")
```

### Date/Time Features

```python
df["day_of_week"] = df["timestamp"].dt.dayofweek
df["hour"] = df["timestamp"].dt.hour
df["month"] = df["timestamp"].dt.month
df["is_weekend"] = df["day_of_week"].isin([5, 6]).astype(int)
df["days_since_signup"] = (df["event_date"] - df["signup_date"]).dt.days
```

### Text Features (Basic)

```python
# Simple text features without NLP
df["text_length"] = df["description"].str.len()
df["word_count"] = df["description"].str.split().str.len()
df["has_exclamation"] = df["description"].str.contains("!").astype(int)
df["uppercase_ratio"] = df["description"].str.count(r"[A-Z]") / df["text_length"]
```

---

## Method Comparison

| Method | Speed | Captures Interactions | Model-Agnostic | Handles Redundancy |
|--------|-------|---------------------|----------------|--------------------|
| Variance Threshold | ⚡ Fast | ❌ No | ✅ Yes | ❌ No |
| Correlation Filter | ⚡ Fast | ❌ No | ✅ Yes | ✅ Yes |
| Mutual Information | ⚡ Fast | ✅ Yes (non-linear) | ✅ Yes | ❌ No |
| RFE | 🐢 Slow | ✅ Yes | ❌ No | ✅ Yes |
| L1 (Lasso) | ⚡ Fast | ⚠️ Partial | ❌ No | ✅ Yes |
| Permutation Importance | 🐢 Medium | ✅ Yes | ✅ Yes | ❌ Partial |
| Tree Importance | ⚡ Fast | ✅ Yes | ❌ Trees only | ❌ Biased |

---

## Key Takeaways

- Feature selection reduces overfitting, speeds up training, and improves interpretability — it is not optional for wide datasets.
- Filter methods (variance, correlation, mutual information) are fast and model-agnostic — use them as a first pass to eliminate clearly useless features.
- Wrapper methods (RFE) evaluate feature subsets by training models — more accurate but slower; use RFECV to find the optimal number of features.
- Permutation importance is the most reliable importance measure — it is model-agnostic and unbiased, unlike tree-based impurity importance.
- Target encoding is powerful for high-cardinality categoricals but must be computed within cross-validation folds to prevent target leakage.
- Feature engineering (interactions, ratios, binning, date decomposition) often delivers more value than algorithm tuning — domain knowledge is the strongest lever.
