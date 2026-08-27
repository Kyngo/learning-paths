---
title: "Hyperparameter Tuning"
weight: 8
---

# Hyperparameter Tuning

Hyperparameters are settings that control the learning process — they are not learned from data. Choosing the right hyperparameters can be the difference between a mediocre model and a production-ready one. This section covers systematic approaches to finding optimal configurations.

---

## Parameters vs Hyperparameters

| Type | Set By | Examples |
|------|--------|---------|
| **Parameters** | Learned from data during training | Linear regression weights, tree split thresholds |
| **Hyperparameters** | Set before training begins | Learning rate, max_depth, C, n_estimators, kernel |

Parameters are optimised by the algorithm (e.g., gradient descent minimises the loss). Hyperparameters are optimised by you — using the techniques in this section.

---

## Grid Search

Exhaustively evaluates every combination in a predefined grid of hyperparameter values.

```python
from sklearn.model_selection import GridSearchCV
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

param_grid = {
    "n_estimators": [50, 100, 200],
    "max_depth": [5, 10, 20, None],
    "min_samples_split": [2, 5, 10],
    "min_samples_leaf": [1, 2, 4]
}

grid = GridSearchCV(
    RandomForestClassifier(random_state=42),
    param_grid,
    cv=5,
    scoring="f1",
    n_jobs=-1,
    verbose=1
)
grid.fit(X_train, y_train)

print(f"Best params: {grid.best_params_}")
print(f"Best CV F1:  {grid.best_score_:.4f}")
print(f"Test F1:     {grid.score(X_test, y_test):.4f}")
```

### Limitations

The grid above has 3 × 4 × 3 × 3 = 108 combinations, each evaluated with 5-fold CV = 540 model fits. Adding more values or hyperparameters causes a combinatorial explosion.

| Grid Size | 5-Fold CV Fits |
|-----------|---------------|
| 3 × 3 | 45 |
| 4 × 4 × 4 | 320 |
| 5 × 5 × 5 × 5 | 3,125 |

---

## Random Search

Samples hyperparameter values randomly from specified distributions. Often more efficient than grid search because it explores more of the search space.

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import randint, uniform

param_distributions = {
    "n_estimators": randint(50, 500),
    "max_depth": randint(3, 30),
    "min_samples_split": randint(2, 20),
    "min_samples_leaf": randint(1, 10),
    "max_features": uniform(0.1, 0.9)
}

random_search = RandomizedSearchCV(
    RandomForestClassifier(random_state=42),
    param_distributions,
    n_iter=100,            # Number of random combinations to try
    cv=5,
    scoring="f1",
    n_jobs=-1,
    random_state=42,
    verbose=1
)
random_search.fit(X_train, y_train)

print(f"Best params: {random_search.best_params_}")
print(f"Best CV F1:  {random_search.best_score_:.4f}")
```

### Why Random Search Beats Grid Search

With grid search, if one hyperparameter matters much more than another, you waste evaluations testing every value of the unimportant one. Random search allocates trials more evenly across all hyperparameters.

Bergstra & Bengio (2012) showed that random search with 60 trials finds a configuration as good as or better than grid search with far more evaluations for most problems.

---

## Bayesian Optimisation with Optuna

Bayesian optimisation uses past trial results to build a probabilistic model of the objective function, then samples promising configurations. It is far more sample-efficient than random search.

### Optuna

Optuna is the most popular Bayesian hyperparameter optimisation library in the Python ecosystem.

```python
import optuna
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score

def objective(trial):
    params = {
        "n_estimators": trial.suggest_int("n_estimators", 50, 500),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3, log=True),
        "max_depth": trial.suggest_int("max_depth", 3, 10),
        "subsample": trial.suggest_float("subsample", 0.5, 1.0),
        "min_samples_split": trial.suggest_int("min_samples_split", 2, 20),
        "min_samples_leaf": trial.suggest_int("min_samples_leaf", 1, 10),
    }

    model = GradientBoostingClassifier(**params, random_state=42)
    scores = cross_val_score(model, X_train, y_train, cv=5, scoring="f1", n_jobs=-1)
    return scores.mean()

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=100, show_progress_bar=True)

print(f"Best params: {study.best_params}")
print(f"Best CV F1:  {study.best_value:.4f}")
```

### Optuna Features

| Feature | Description |
|---------|------------|
| `suggest_int` | Integer parameter |
| `suggest_float` | Float parameter (optional `log=True` for log-uniform) |
| `suggest_categorical` | Categorical choices |
| Pruning | Early-stop unpromising trials (saves compute) |
| Visualisation | `optuna.visualization.plot_optimization_history(study)` |
| Persistence | Save/resume studies with SQLite or PostgreSQL |

### Visualising the Search

```python
from optuna.visualization import (
    plot_optimization_history,
    plot_param_importances,
    plot_contour
)

# How the best value improves over trials
plot_optimization_history(study).show()

# Which hyperparameters matter most
plot_param_importances(study).show()

# Interaction between two hyperparameters
plot_contour(study, params=["learning_rate", "max_depth"]).show()
```

---

## Early Stopping

For iterative algorithms (gradient boosting, neural networks), early stopping monitors validation performance and halts training when it stops improving. This prevents overfitting without specifying the exact number of iterations.

```python
from xgboost import XGBClassifier

xgb = XGBClassifier(
    n_estimators=1000,          # Set high — early stopping will find the right value
    learning_rate=0.05,
    max_depth=6,
    random_state=42,
    early_stopping_rounds=50,   # Stop if no improvement in 50 rounds
    eval_metric="logloss"
)

xgb.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    verbose=False
)

print(f"Best iteration: {xgb.best_iteration}")
print(f"Test accuracy:  {xgb.score(X_test, y_test):.4f}")
```

### LightGBM Early Stopping

```python
from lightgbm import LGBMClassifier, early_stopping

lgbm = LGBMClassifier(
    n_estimators=1000,
    learning_rate=0.05,
    num_leaves=31,
    random_state=42,
    verbose=-1
)

lgbm.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    callbacks=[early_stopping(stopping_rounds=50)]
)
```

---

## Model Selection Strategy

### Recommended Workflow

```text
1. Establish baseline       → Simple model (logistic regression, dummy classifier)
2. Try default models       → RF, XGBoost, LightGBM with defaults
3. Random search            → 50-100 iterations on the best 1-2 candidates
4. Bayesian optimisation    → Refine the top candidate with Optuna (100-200 trials)
5. Final evaluation         → Retrain on full train+val, evaluate on held-out test
```

### Comparison Table

| Method | Efficiency | Setup Effort | Best For |
|--------|-----------|-------------|----------|
| Grid Search | Low (exhaustive) | Low (just define grid) | Few hyperparameters (1-3) |
| Random Search | Medium | Low | First exploration pass |
| Bayesian (Optuna) | High (learns from trials) | Medium | Final optimisation |
| Manual Tuning | Variable | High (requires expertise) | Quick experiments |

### Common Hyperparameter Ranges

| Algorithm | Hyperparameter | Typical Range |
|-----------|---------------|---------------|
| Random Forest | `n_estimators` | 100–1000 |
| Random Forest | `max_depth` | 5–30 or None |
| XGBoost / LightGBM | `learning_rate` | 0.01–0.3 |
| XGBoost / LightGBM | `n_estimators` | 100–2000 (with early stopping) |
| XGBoost / LightGBM | `max_depth` | 3–12 |
| XGBoost / LightGBM | `subsample` | 0.5–1.0 |
| XGBoost / LightGBM | `colsample_bytree` | 0.3–1.0 |
| SVM | `C` | 0.001–1000 (log scale) |
| SVM | `gamma` | 0.0001–10 (log scale) |
| Logistic Regression | `C` | 0.001–100 (log scale) |

### Avoiding Overfitting the Validation Set

Running hundreds of trials on the same validation set can lead to overfitting the validation data. Mitigations:

- Use **nested cross-validation**: outer loop for model evaluation, inner loop for hyperparameter tuning
- Keep a final **hold-out test set** that is never used during tuning
- Report both CV score and test score — a large gap indicates overfitting the search

```python
from sklearn.model_selection import cross_val_score

# Nested CV: the inner CV is inside GridSearchCV, the outer CV evaluates it
grid = GridSearchCV(RandomForestClassifier(random_state=42), param_grid, cv=3, scoring="f1")
nested_scores = cross_val_score(grid, X, y, cv=5, scoring="f1")
print(f"Nested CV F1: {nested_scores.mean():.4f} ± {nested_scores.std():.4f}")
```

---

## Key Takeaways

- Hyperparameters control the learning process and must be tuned separately from model parameters — never tune on the test set.
- Grid search is exhaustive but scales poorly; random search is more efficient for the same budget and should be your default starting point.
- Bayesian optimisation (Optuna) learns from past trials and is the most sample-efficient method — use it for final optimisation on your best candidate model.
- Early stopping prevents overfitting in iterative algorithms and effectively tunes the number of iterations automatically.
- Follow a structured workflow: baseline → defaults → random search → Bayesian optimisation → final test evaluation.
- Nested cross-validation gives the most honest performance estimate when tuning is part of the evaluation.
