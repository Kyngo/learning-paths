---
title: "Time Series"
weight: 10
---

# Time Series

Time series data has a temporal ordering that standard ML methods ignore. Forecasting future values, detecting seasonality, and engineering lag features require specialised techniques. This section covers classical time series analysis and how to apply ML models to temporal data correctly.

---

## Time Series Fundamentals

A time series is a sequence of observations ordered by time: `y₁, y₂, ..., yₜ`.

### Components

Every time series can be decomposed into:

| Component | Description | Example |
|-----------|------------|---------|
| **Trend** | Long-term increase or decrease | Growing user base |
| **Seasonality** | Repeating patterns at fixed intervals | Higher sales in December |
| **Cyclical** | Repeating patterns not at fixed intervals | Economic cycles |
| **Residual** | Random noise after removing trend and seasonality | Unpredictable variation |

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from statsmodels.tsa.seasonal import seasonal_decompose

# Generate example time series
np.random.seed(42)
dates = pd.date_range("2020-01-01", periods=365 * 3, freq="D")
trend = np.linspace(100, 200, len(dates))
seasonal = 20 * np.sin(2 * np.pi * np.arange(len(dates)) / 365)
noise = np.random.normal(0, 5, len(dates))
y = trend + seasonal + noise

ts = pd.Series(y, index=dates)

# Decompose
result = seasonal_decompose(ts, model="additive", period=365)
result.plot()
plt.tight_layout()
plt.show()
```

---

## Stationarity

A time series is **stationary** if its statistical properties (mean, variance, autocorrelation) do not change over time. Most classical forecasting methods require stationarity.

### Testing for Stationarity

The **Augmented Dickey-Fuller (ADF)** test:
- H₀: The series has a unit root (non-stationary)
- H₁: The series is stationary
- If p-value < 0.05, reject H₀ → series is stationary

```python
from statsmodels.tsa.stattools import adfuller

result = adfuller(ts)
print(f"ADF Statistic: {result[0]:.4f}")
print(f"p-value:       {result[1]:.4f}")
print(f"Stationary:    {'Yes' if result[1] < 0.05 else 'No'}")
```

### Making a Series Stationary

| Technique | Effect | Code |
|-----------|--------|------|
| Differencing | Removes trend | `ts.diff()` |
| Second differencing | Removes quadratic trend | `ts.diff().diff()` |
| Log transform | Stabilises variance | `np.log(ts)` |
| Seasonal differencing | Removes seasonality | `ts.diff(periods=365)` |

```python
# First-order differencing
ts_diff = ts.diff().dropna()

# Check stationarity after differencing
result = adfuller(ts_diff)
print(f"After differencing — p-value: {result[1]:.4f}")
```

---

## ARIMA

**AutoRegressive Integrated Moving Average** — the classical forecasting model.

| Component | Symbol | Meaning |
|-----------|--------|---------|
| AR(p) | p | Number of lag observations (autoregressive terms) |
| I(d) | d | Degree of differencing for stationarity |
| MA(q) | q | Size of the moving average window |

### Fitting ARIMA

```python
from statsmodels.tsa.arima.model import ARIMA

# Split chronologically
train = ts[:"2022-06-30"]
test = ts["2022-07-01":]

model = ARIMA(train, order=(5, 1, 2))
fitted = model.fit()
print(fitted.summary())

# Forecast
forecast = fitted.forecast(steps=len(test))

plt.figure(figsize=(12, 4))
plt.plot(train[-90:], label="Train")
plt.plot(test, label="Actual")
plt.plot(forecast, label="Forecast", linestyle="--")
plt.legend()
plt.title("ARIMA Forecast")
plt.show()
```

### SARIMA (Seasonal ARIMA)

Extends ARIMA with seasonal terms `(P, D, Q, s)`:

```python
from statsmodels.tsa.statespace.sarimax import SARIMAX

model = SARIMAX(train, order=(1, 1, 1), seasonal_order=(1, 1, 1, 365))
fitted = model.fit(disp=False)
forecast = fitted.forecast(steps=len(test))
```

### Auto ARIMA

Automatically selects the best (p, d, q) parameters:

```python
from pmdarima import auto_arima

auto_model = auto_arima(
    train,
    seasonal=True,
    m=365,
    suppress_warnings=True,
    stepwise=True
)
print(auto_model.summary())
forecast = auto_model.predict(n_periods=len(test))
```

---

## Feature Engineering for Time Series

When using ML models (random forest, XGBoost) for time series, you need to manually create temporal features.

### Lag Features

```python
df = pd.DataFrame({"date": dates, "value": y}).set_index("date")

# Lag features
for lag in [1, 7, 14, 30]:
    df[f"lag_{lag}"] = df["value"].shift(lag)

# Rolling statistics
for window in [7, 14, 30]:
    df[f"rolling_mean_{window}"] = df["value"].rolling(window).mean()
    df[f"rolling_std_{window}"] = df["value"].rolling(window).std()

# Expanding mean
df["expanding_mean"] = df["value"].expanding().mean()

# Drop rows with NaN from lagging
df = df.dropna()
```

### Calendar Features

```python
df["day_of_week"] = df.index.dayofweek
df["day_of_month"] = df.index.day
df["month"] = df.index.month
df["quarter"] = df.index.quarter
df["day_of_year"] = df.index.dayofyear
df["is_weekend"] = (df.index.dayofweek >= 5).astype(int)
df["is_month_start"] = df.index.is_month_start.astype(int)
df["is_month_end"] = df.index.is_month_end.astype(int)
```

### Cyclical Encoding

For periodic features, encode them as sine/cosine pairs to preserve the cyclical relationship:

```python
df["month_sin"] = np.sin(2 * np.pi * df["month"] / 12)
df["month_cos"] = np.cos(2 * np.pi * df["month"] / 12)
df["dow_sin"] = np.sin(2 * np.pi * df["day_of_week"] / 7)
df["dow_cos"] = np.cos(2 * np.pi * df["day_of_week"] / 7)
```

### ML Model on Time Series Features

```python
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.metrics import mean_absolute_error

target = "value"
features = [c for c in df.columns if c != target]

# Chronological split — never shuffle time series
split_idx = int(len(df) * 0.8)
X_train, X_test = df[features].iloc[:split_idx], df[features].iloc[split_idx:]
y_train, y_test = df[target].iloc[:split_idx], df[target].iloc[split_idx:]

model = GradientBoostingRegressor(n_estimators=200, max_depth=5, learning_rate=0.1)
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
print(f"MAE: {mean_absolute_error(y_test, y_pred):.2f}")
```

---

## Prophet

Facebook's Prophet is designed for business time series with strong seasonal effects and missing data.

```python
from prophet import Prophet

# Prophet requires columns named 'ds' (date) and 'y' (value)
prophet_df = pd.DataFrame({"ds": dates, "y": y})

train_prophet = prophet_df[prophet_df["ds"] <= "2022-06-30"]
test_prophet = prophet_df[prophet_df["ds"] > "2022-06-30"]

model = Prophet(yearly_seasonality=True, weekly_seasonality=True, daily_seasonality=False)
model.fit(train_prophet)

future = model.make_future_dataframe(periods=len(test_prophet))
forecast = model.predict(future)

model.plot(forecast)
plt.title("Prophet Forecast")
plt.show()

model.plot_components(forecast)
plt.show()
```

### When to Use Prophet

| ✅ Good Fit | ❌ Poor Fit |
|------------|-----------|
| Business metrics (revenue, traffic) | High-frequency data (sub-hourly) |
| Strong seasonal patterns | Complex multivariate dependencies |
| Missing data and outliers present | Short time series (< 2 seasons) |
| Non-technical stakeholders need interpretable output | Need fine-grained control over the model |

---

## Time Series Cross-Validation

Standard K-Fold shuffles data and breaks temporal ordering. Time series requires **expanding window** or **sliding window** validation.

```python
from sklearn.model_selection import TimeSeriesSplit
import matplotlib.pyplot as plt

tscv = TimeSeriesSplit(n_splits=5)

fig, ax = plt.subplots(figsize=(12, 4))
for i, (train_idx, test_idx) in enumerate(tscv.split(df)):
    ax.plot(train_idx, [i] * len(train_idx), "b.", markersize=1)
    ax.plot(test_idx, [i] * len(test_idx), "r.", markersize=1)
ax.set_ylabel("CV Fold")
ax.set_xlabel("Sample Index")
ax.set_title("Time Series Cross-Validation")
plt.show()
```

### Custom Expanding Window CV

```python
from sklearn.model_selection import cross_val_score

tscv = TimeSeriesSplit(n_splits=5, gap=7)  # 7-day gap to avoid data leakage

scores = cross_val_score(
    GradientBoostingRegressor(n_estimators=200, max_depth=5),
    df[features], df[target],
    cv=tscv,
    scoring="neg_mean_absolute_error"
)
print(f"CV MAE: {-scores.mean():.2f} ± {scores.std():.2f}")
```

### Evaluation Metrics for Forecasting

| Metric | Formula | Notes |
|--------|---------|-------|
| MAE | Σ\|yᵢ - ŷᵢ\| / n | Easy to interpret |
| RMSE | √(Σ(yᵢ - ŷᵢ)² / n) | Penalises large errors |
| MAPE | Σ(\|yᵢ - ŷᵢ\| / \|yᵢ\|) / n × 100 | Percentage error, undefined for y=0 |
| sMAPE | Σ(2\|yᵢ - ŷᵢ\| / (\|yᵢ\| + \|ŷᵢ\|)) / n × 100 | Symmetric, bounded |

---

## Key Takeaways

- Time series data requires respecting temporal order — never shuffle, always split chronologically, and use `TimeSeriesSplit` for cross-validation.
- Stationarity is required by ARIMA and many classical methods — use differencing and transformations to achieve it, and test with the ADF test.
- ARIMA models the autocorrelation structure directly; SARIMA adds seasonal components. Use `auto_arima` for automated order selection.
- Lag features, rolling statistics, and calendar features transform time series into tabular data suitable for gradient boosting and other ML models.
- Prophet is effective for business forecasting with strong seasonality, missing data, and interpretability requirements.
- Encode cyclical features (month, day of week) with sine/cosine pairs rather than raw integers to preserve the periodic relationship.
