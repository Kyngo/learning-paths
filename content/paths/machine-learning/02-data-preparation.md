---
title: "Data Preparation"
weight: 2
---

# Data Preparation

Data preparation consumes 60–80% of a typical ML project's time. A model trained on poorly prepared data will produce poor results regardless of how sophisticated the algorithm is. This section covers the essential transformations between raw data and model-ready features.

---

## Exploratory Data Analysis (EDA)

Before any transformation, understand your data:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

df = pd.read_csv("housing.csv")

# Shape, types, memory
print(df.shape)
print(df.dtypes)
print(df.describe())

# Missing values
print(df.isnull().sum().sort_values(ascending=False))

# Distribution of target
df["price"].hist(bins=50)
plt.xlabel("Price")
plt.ylabel("Count")
plt.title("Target Distribution")
plt.show()

# Correlation matrix
corr = df.select_dtypes(include=[np.number]).corr()
sns.heatmap(corr, annot=True, fmt=".2f", cmap="coolwarm", center=0)
plt.title("Feature Correlations")
plt.tight_layout()
plt.show()
```

### EDA Checklist

| Check | What to Look For |
|-------|-----------------|
| Shape | Number of rows and features |
| Types | Numeric vs categorical vs datetime |
| Missing values | Percentage per column, patterns (MCAR/MAR/MNAR) |
| Distributions | Skew, outliers, multimodality |
| Target balance | Class imbalance for classification |
| Correlations | Redundant features, leakage signals |
| Duplicates | Exact or near-duplicate rows |

---

## Handling Missing Data

Missing data is nearly universal. The strategy depends on why the data is missing.

| Type | Meaning | Example | Strategy |
|------|---------|---------|----------|
| MCAR | Missing completely at random | Sensor glitch | Drop or impute |
| MAR | Missing depends on observed data | High-income people skip income question less | Impute using related features |
| MNAR | Missing depends on the missing value itself | Very sick patients miss follow-up | Model the missingness, domain expertise |

### Imputation Strategies

```python
from sklearn.impute import SimpleImputer, KNNImputer

# Numerical: median (robust to outliers)
num_imputer = SimpleImputer(strategy="median")
df[num_cols] = num_imputer.fit_transform(df[num_cols])

# Categorical: most frequent
cat_imputer = SimpleImputer(strategy="most_frequent")
df[cat_cols] = cat_imputer.fit_transform(df[cat_cols])

# KNN imputation (uses similar rows to fill values)
knn_imputer = KNNImputer(n_neighbors=5)
df[num_cols] = knn_imputer.fit_transform(df[num_cols])
```

### Missing Indicator Features

Sometimes the fact that a value is missing carries information:

```python
from sklearn.impute import SimpleImputer, MissingIndicator

# Create binary columns: 1 if the original value was missing
indicator = MissingIndicator()
missing_flags = indicator.fit_transform(df[num_cols])
```

**Rule of thumb:** If a column has more than 50% missing values, consider dropping it unless domain knowledge justifies imputation.

---

## Encoding Categorical Variables

Models need numbers. Categorical features must be encoded.

### Ordinal Encoding

For features with a natural order:

```python
from sklearn.preprocessing import OrdinalEncoder

# size: S < M < L < XL
encoder = OrdinalEncoder(categories=[["S", "M", "L", "XL"]])
df["size_encoded"] = encoder.fit_transform(df[["size"]])
```

### One-Hot Encoding

For nominal categories (no inherent order):

```python
from sklearn.preprocessing import OneHotEncoder

encoder = OneHotEncoder(sparse_output=False, drop="first", handle_unknown="ignore")
encoded = encoder.fit_transform(df[["city"]])
```

| Parameter | Purpose |
|-----------|---------|
| `drop="first"` | Avoids multicollinearity (the dummy variable trap) |
| `handle_unknown="ignore"` | Returns all zeros for categories not seen during training |
| `sparse_output=False` | Returns a dense array instead of sparse matrix |

### When to Use Which

| Encoding | Use When | Watch Out For |
|----------|----------|--------------|
| Ordinal | Natural order exists (size, rating, education level) | Implies numeric distance between categories |
| One-Hot | Nominal categories, low cardinality (< 20) | High cardinality explodes feature space |
| Target Encoding | High cardinality (city, ZIP code, product ID) | Risk of target leakage — use with cross-validation |
| Frequency Encoding | Category frequency is informative | Collisions when two categories have the same frequency |
| Binary Encoding | High cardinality, tree-based models | Less interpretable |

---

## Feature Scaling

Many algorithms are sensitive to feature magnitude — gradient descent converges faster, distance-based methods work correctly, and regularisation penalises features fairly only when features are on comparable scales.

### Standardisation (Z-score)

Transforms features to mean=0, std=1:

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)  # Use train statistics, never fit on test
```

### Min-Max Normalisation

Scales features to [0, 1]:

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)
```

### Robust Scaling

Uses median and IQR — resistant to outliers:

```python
from sklearn.preprocessing import RobustScaler

scaler = RobustScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)
```

### When Scaling Matters

| Algorithm | Needs Scaling? | Why |
|-----------|---------------|-----|
| Linear/Logistic Regression | ✅ Yes | Regularisation penalises large coefficients |
| SVM | ✅ Yes | Distance-based kernel computations |
| K-NN | ✅ Yes | Distance-based predictions |
| K-Means | ✅ Yes | Distance-based cluster assignment |
| PCA | ✅ Yes | Maximises variance — dominated by large-scale features |
| Decision Trees | ❌ No | Splits on thresholds, scale-invariant |
| Random Forest | ❌ No | Ensemble of trees |
| Gradient Boosting | ❌ No | Tree-based, scale-invariant |

---

## Train / Validation / Test Splits

### Simple Split

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y  # stratify for classification
)
```

### Stratified Splits

For imbalanced classification, `stratify=y` ensures each split has the same class proportions as the original dataset.

### Time-Based Splits

For time series or temporal data, **never shuffle**. Future data must not leak into training:

```python
# Sort by date, split chronologically
df = df.sort_values("date")
split_idx = int(len(df) * 0.8)
train = df.iloc[:split_idx]
test = df.iloc[split_idx:]
```

### Data Leakage

Data leakage occurs when information from outside the training set influences the model during training. It is the most common source of overly optimistic performance estimates.

| Leakage Type | Example | Prevention |
|-------------|---------|------------|
| Target leakage | Feature computed from the target variable | Audit feature provenance |
| Train-test contamination | Fitting scaler on full dataset before splitting | Always `fit` on training data only |
| Temporal leakage | Using future data to predict the past | Respect time ordering in splits |
| Group leakage | Same patient in train and test sets | Use `GroupKFold` or `GroupShuffleSplit` |

```python
# WRONG — fits scaler on all data (leaks test statistics into training)
scaler.fit(X)
X_train_scaled = scaler.transform(X_train)
X_test_scaled = scaler.transform(X_test)

# CORRECT — fits scaler only on training data
scaler.fit(X_train)
X_train_scaled = scaler.transform(X_train)
X_test_scaled = scaler.transform(X_test)
```

---

## Handling Outliers

Outliers can distort model training, particularly for linear models and distance-based algorithms.

### Detection

```python
# IQR method
Q1 = df["feature"].quantile(0.25)
Q3 = df["feature"].quantile(0.75)
IQR = Q3 - Q1
lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR
outliers = df[(df["feature"] < lower) | (df["feature"] > upper)]

# Z-score method
from scipy import stats
z_scores = np.abs(stats.zscore(df[num_cols]))
outliers = (z_scores > 3).any(axis=1)
```

### Treatment

| Strategy | When to Use |
|----------|------------|
| Remove | Clearly erroneous data (negative ages, impossible values) |
| Cap/clip | Extreme but valid values (winsorise to 1st/99th percentile) |
| Transform | Log or square root to reduce skew |
| Keep | Tree-based models handle outliers naturally |

```python
# Winsorisation: clip to 1st and 99th percentiles
lower = df["income"].quantile(0.01)
upper = df["income"].quantile(0.99)
df["income_clipped"] = df["income"].clip(lower, upper)

# Log transform for right-skewed features
df["log_income"] = np.log1p(df["income"])  # log1p handles zeros
```

---

## Key Takeaways

- Exploratory data analysis is not optional — understand distributions, missing patterns, correlations, and class balance before modelling.
- Choose imputation strategy based on missingness type: median for numeric (robust to outliers), mode for categorical, KNN for leveraging feature relationships.
- One-hot encode nominal categories; use ordinal encoding only when a natural order exists. Beware high cardinality — consider target encoding with cross-validation.
- Scale features for algorithms sensitive to magnitude (linear models, SVM, KNN, PCA) — always fit the scaler on training data only.
- Data leakage is the most dangerous pitfall in ML — it produces unrealistically good validation scores that collapse in production.
- Outlier handling depends on the algorithm: tree-based models are robust, linear and distance-based models are not.
