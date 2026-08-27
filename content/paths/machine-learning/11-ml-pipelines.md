---
title: "ML Pipelines & scikit-learn"
weight: 11
---

# ML Pipelines & scikit-learn

Production ML code is not a sequence of Jupyter cells — it is a reproducible pipeline that transforms raw data into predictions without manual intervention. Scikit-learn's `Pipeline` and `ColumnTransformer` are the building blocks for clean, leak-free, serialisable ML workflows.

---

## Why Pipelines Matter

Without pipelines, preprocessing and modelling are separate steps prone to errors:

| Problem | Consequence |
|---------|------------|
| Fitting scaler on full data before splitting | Data leakage |
| Forgetting a transform step in production | Wrong predictions |
| Different code paths for training and inference | Silent bugs |
| Manual step ordering | Impossible to reproduce |

A `Pipeline` chains transforms and a final estimator into a single object that implements `fit()`, `predict()`, and `score()`.

---

## Basic Pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split, cross_val_score

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("classifier", LogisticRegression(max_iter=5000, random_state=42))
])

# The pipeline handles fit_transform on scaler, then fit on classifier
pipe.fit(X_train, y_train)
print(f"Test accuracy: {pipe.score(X_test, y_test):.4f}")

# Cross-validation respects the pipeline — no leakage
scores = cross_val_score(pipe, X, y, cv=5, scoring="accuracy")
print(f"CV accuracy: {scores.mean():.4f} ± {scores.std():.4f}")
```

### make_pipeline Shorthand

When you do not need custom step names:

```python
from sklearn.pipeline import make_pipeline

pipe = make_pipeline(StandardScaler(), LogisticRegression(max_iter=5000))
```

---

## ColumnTransformer

Real datasets have mixed types — numeric columns need scaling while categorical columns need encoding. `ColumnTransformer` applies different transformations to different column subsets.

```python
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.ensemble import RandomForestClassifier
import pandas as pd

# Example mixed-type data
data = {
    "age": [25, 30, None, 45, 35],
    "income": [50000, 60000, 45000, None, 70000],
    "city": ["London", "Paris", "London", "Berlin", "Paris"],
    "education": ["BSc", "MSc", "PhD", "BSc", "MSc"],
    "churned": [0, 0, 1, 1, 0]
}
df = pd.DataFrame(data)

numeric_features = ["age", "income"]
categorical_features = ["city", "education"]

preprocessor = ColumnTransformer(
    transformers=[
        ("num", Pipeline([
            ("imputer", SimpleImputer(strategy="median")),
            ("scaler", StandardScaler())
        ]), numeric_features),
        ("cat", Pipeline([
            ("imputer", SimpleImputer(strategy="most_frequent")),
            ("encoder", OneHotEncoder(drop="first", handle_unknown="ignore"))
        ]), categorical_features)
    ],
    remainder="drop"  # Drop columns not listed (or "passthrough" to keep them)
)

# Full pipeline: preprocess → model
full_pipe = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", RandomForestClassifier(n_estimators=100, random_state=42))
])

X = df.drop("churned", axis=1)
y = df["churned"]

full_pipe.fit(X, y)
predictions = full_pipe.predict(X)
```

### Accessing Transformed Feature Names

```python
# After fitting, retrieve the output column names
feature_names = full_pipe.named_steps["preprocessor"].get_feature_names_out()
print(feature_names)
```

---

## Custom Transformers

When built-in transformers are not enough, create your own by implementing `fit()` and `transform()`.

### Using FunctionTransformer

For stateless transformations:

```python
from sklearn.preprocessing import FunctionTransformer
import numpy as np

log_transformer = FunctionTransformer(np.log1p, validate=True)

pipe = make_pipeline(log_transformer, StandardScaler(), LogisticRegression())
```

### Full Custom Transformer

For stateful transformations (that learn from training data):

```python
from sklearn.base import BaseEstimator, TransformerMixin

class OutlierClipper(BaseEstimator, TransformerMixin):
    """Clips features to [Q1 - 1.5*IQR, Q3 + 1.5*IQR] computed from training data."""

    def __init__(self, factor=1.5):
        self.factor = factor

    def fit(self, X, y=None):
        Q1 = np.percentile(X, 25, axis=0)
        Q3 = np.percentile(X, 75, axis=0)
        IQR = Q3 - Q1
        self.lower_ = Q1 - self.factor * IQR
        self.upper_ = Q3 + self.factor * IQR
        return self

    def transform(self, X):
        return np.clip(X, self.lower_, self.upper_)

# Use it in a pipeline
pipe = Pipeline([
    ("clipper", OutlierClipper(factor=1.5)),
    ("scaler", StandardScaler()),
    ("model", RandomForestClassifier())
])
```

### Rules for Custom Transformers

| Rule | Why |
|------|-----|
| Inherit from `BaseEstimator` and `TransformerMixin` | Provides `get_params()`, `set_params()`, and `fit_transform()` |
| Store learned state in attributes ending with `_` | Convention for fitted attributes (e.g., `self.lower_`) |
| `fit()` must return `self` | Enables method chaining |
| Never learn from `y` in a transformer (unless it is a target encoder) | Prevents target leakage |
| `transform()` must accept the same shape as `fit()` | Consistency between training and inference |

---

## Hyperparameter Tuning with Pipelines

Pipeline step parameters are accessed with `stepname__parameter` syntax:

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    "preprocessor__num__imputer__strategy": ["mean", "median"],
    "classifier__n_estimators": [100, 200],
    "classifier__max_depth": [5, 10, None]
}

grid = GridSearchCV(full_pipe, param_grid, cv=5, scoring="f1", n_jobs=-1)
grid.fit(X, y)
print(f"Best params: {grid.best_params_}")
```

---

## Serialisation

### Joblib (Recommended for scikit-learn)

```python
import joblib

# Save the entire pipeline
joblib.dump(full_pipe, "model_pipeline.joblib")

# Load and predict
loaded_pipe = joblib.load("model_pipeline.joblib")
predictions = loaded_pipe.predict(new_data)
```

### Pickle

```python
import pickle

with open("model_pipeline.pkl", "wb") as f:
    pickle.dump(full_pipe, f)

with open("model_pipeline.pkl", "rb") as f:
    loaded_pipe = pickle.load(f)
```

### Serialisation Best Practices

| Practice | Why |
|----------|-----|
| Save the entire pipeline, not just the model | Ensures preprocessing is included |
| Version your scikit-learn when saving | Pickle compatibility breaks across versions |
| Save alongside metadata (features, target, metrics) | Reproducibility |
| Never unpickle untrusted files | Security risk (arbitrary code execution) |

---

## Reproducibility Checklist

| Item | How |
|------|-----|
| Random seeds | Set `random_state` on all estimators and splitters |
| Dependency versions | Pin scikit-learn, numpy, pandas versions in `requirements.txt` |
| Data versioning | Hash your training data or use DVC |
| Pipeline definition | Save pipeline config (parameters, step order) alongside the model |
| Feature names | Store input feature names and their expected order |
| Training metrics | Log CV scores, test scores, confusion matrix |

```python
# Example: saving metadata alongside the model
import json
from datetime import datetime

metadata = {
    "trained_at": datetime.now().isoformat(),
    "sklearn_version": sklearn.__version__,
    "features": list(X.columns),
    "cv_score": float(scores.mean()),
    "test_score": float(full_pipe.score(X_test, y_test)),
    "params": full_pipe.get_params()
}

with open("model_metadata.json", "w") as f:
    json.dump(metadata, f, indent=2, default=str)
```

---

## Complete Example

```python
import pandas as pd
import numpy as np
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score, train_test_split
import joblib

# Define column types
numeric_features = ["age", "income", "account_age_days"]
categorical_features = ["country", "plan_type"]

# Build preprocessor
preprocessor = ColumnTransformer([
    ("num", Pipeline([
        ("imputer", SimpleImputer(strategy="median")),
        ("scaler", StandardScaler())
    ]), numeric_features),
    ("cat", Pipeline([
        ("imputer", SimpleImputer(strategy="most_frequent")),
        ("encoder", OneHotEncoder(drop="first", handle_unknown="ignore"))
    ]), categorical_features)
])

# Full pipeline
pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", GradientBoostingClassifier(
        n_estimators=200, max_depth=5, learning_rate=0.1, random_state=42
    ))
])

# Train and evaluate
pipeline.fit(X_train, y_train)
scores = cross_val_score(pipeline, X_train, y_train, cv=5, scoring="f1")
print(f"CV F1: {scores.mean():.4f} ± {scores.std():.4f}")
print(f"Test F1: {pipeline.score(X_test, y_test):.4f}")

# Save
joblib.dump(pipeline, "churn_pipeline.joblib")
```

---

## Key Takeaways

- Pipelines chain preprocessing and modelling into a single object — they prevent data leakage by ensuring transforms are fitted only on training data during cross-validation.
- `ColumnTransformer` applies different transformations to different column types (numeric, categorical) — this is how production pipelines handle real-world mixed-type data.
- Custom transformers inherit from `BaseEstimator` and `TransformerMixin` — store learned state in `_`-suffixed attributes and always return `self` from `fit()`.
- Serialise the entire pipeline (not just the model) using joblib — this guarantees preprocessing is included in the saved artefact.
- Use `stepname__parameter` syntax to tune hyperparameters of any step within a pipeline using `GridSearchCV` or `RandomizedSearchCV`.
- Reproducibility requires pinned random seeds, versioned dependencies, data hashes, and saved metadata alongside the model file.
