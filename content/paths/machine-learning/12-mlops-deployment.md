---
title: "MLOps & Deployment"
weight: 12
---

# MLOps & Deployment

A model that exists only in a notebook delivers zero value. MLOps is the set of practices that takes a trained model from experimentation to production — covering experiment tracking, model versioning, serving, monitoring, and handling the inevitable data drift. This section bridges the gap between data science and production engineering.

---

## MLOps Overview

| Stage | Concern | Tools |
|-------|---------|-------|
| Experiment tracking | Log parameters, metrics, artefacts per run | MLflow, Weights & Biases, Neptune |
| Model versioning | Version control for models and datasets | MLflow Model Registry, DVC |
| Testing | Validate model behaviour before deployment | pytest, Great Expectations |
| Serving | Expose the model via an API | Flask, FastAPI, BentoML, SageMaker |
| Monitoring | Detect performance degradation in production | Evidently, Nannyml, custom dashboards |
| Retraining | Update the model when data changes | Scheduled pipelines, trigger-based |

---

## Experiment Tracking with MLflow

MLflow tracks experiments — every training run logs its parameters, metrics, and output artefacts.

### Setup and Basic Logging

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score
from sklearn.metrics import f1_score, accuracy_score

# Set experiment name
mlflow.set_experiment("churn-prediction")

with mlflow.start_run(run_name="gbm-baseline"):
    # Log parameters
    params = {
        "n_estimators": 200,
        "max_depth": 5,
        "learning_rate": 0.1,
        "subsample": 0.8
    }
    mlflow.log_params(params)

    # Train
    model = GradientBoostingClassifier(**params, random_state=42)
    model.fit(X_train, y_train)

    # Evaluate
    y_pred = model.predict(X_test)
    f1 = f1_score(y_test, y_pred)
    acc = accuracy_score(y_test, y_pred)

    cv_scores = cross_val_score(model, X_train, y_train, cv=5, scoring="f1")

    # Log metrics
    mlflow.log_metric("f1_test", f1)
    mlflow.log_metric("accuracy_test", acc)
    mlflow.log_metric("f1_cv_mean", cv_scores.mean())
    mlflow.log_metric("f1_cv_std", cv_scores.std())

    # Log the model
    mlflow.sklearn.log_model(model, "model")

    print(f"F1: {f1:.4f}, Accuracy: {acc:.4f}")
```

### MLflow UI

```bash
# Start the tracking UI
mlflow ui --port 5000

# Open http://localhost:5000 to compare runs
```

### Model Registry

The model registry manages model lifecycle stages: Staging → Production → Archived.

```python
# Register a model
model_uri = f"runs:/{mlflow.active_run().info.run_id}/model"
mlflow.register_model(model_uri, "churn-classifier")

# Load a registered model by stage
model = mlflow.sklearn.load_model("models:/churn-classifier/Production")
```

---

## Model Versioning

### What to Version

| Artefact | Tool | Why |
|----------|------|-----|
| Code | Git | Reproducible pipeline code |
| Data | DVC, Delta Lake | Know exactly what data trained each model |
| Model | MLflow, DVC | Roll back to any previous version |
| Config | Git (params.yaml) | Hyperparameters, feature lists |
| Environment | Docker, conda-lock, pip freeze | Dependency versions |

### DVC for Data Versioning

```bash
# Initialise DVC in a git repo
dvc init

# Track a large dataset
dvc add data/training.csv

# Push data to remote storage (S3, GCS, etc.)
dvc remote add -d myremote s3://my-bucket/dvc
dvc push
```

---

## Serving Models

### Flask (Simple)

```python
# serve.py
from flask import Flask, request, jsonify
import joblib

app = Flask(__name__)
model = joblib.load("churn_pipeline.joblib")

@app.route("/predict", methods=["POST"])
def predict():
    data = request.get_json()
    import pandas as pd
    df = pd.DataFrame([data])
    prediction = model.predict(df)[0]
    probability = model.predict_proba(df)[0].tolist()
    return jsonify({
        "prediction": int(prediction),
        "probability": probability
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

### FastAPI (Production-Grade)

```python
# serve_fastapi.py
from fastapi import FastAPI
from pydantic import BaseModel
import joblib
import pandas as pd

app = FastAPI(title="Churn Prediction API")
model = joblib.load("churn_pipeline.joblib")

class PredictionRequest(BaseModel):
    age: float
    income: float
    account_age_days: int
    country: str
    plan_type: str

class PredictionResponse(BaseModel):
    prediction: int
    probability: list[float]

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    df = pd.DataFrame([request.model_dump()])
    prediction = model.predict(df)[0]
    probability = model.predict_proba(df)[0].tolist()
    return PredictionResponse(prediction=int(prediction), probability=probability)

@app.get("/health")
def health():
    return {"status": "healthy"}
```

```bash
# Run with uvicorn
uvicorn serve_fastapi:app --host 0.0.0.0 --port 8080

# Interactive docs at http://localhost:8080/docs
```

### BentoML (ML-Specific)

```python
import bentoml

# Save model
saved_model = bentoml.sklearn.save_model("churn_model", model)

# Create a service
# service.py
import bentoml
import pandas as pd
from bentoml.io import JSON

model_runner = bentoml.sklearn.get("churn_model:latest").to_runner()
svc = bentoml.Service("churn_service", runners=[model_runner])

@svc.api(input=JSON(), output=JSON())
def predict(input_data: dict) -> dict:
    df = pd.DataFrame([input_data])
    result = model_runner.predict.run(df)
    return {"prediction": int(result[0])}
```

### Comparison

| Framework | Complexity | Auto Docs | Async | ML-Specific | Best For |
|-----------|-----------|-----------|-------|-------------|----------|
| Flask | Low | No | No | No | Prototyping |
| FastAPI | Medium | Yes (OpenAPI) | Yes | No | Production APIs |
| BentoML | Medium | Yes | Yes | Yes | ML-native serving |
| SageMaker | High | Managed | Managed | Yes | AWS-native deployment |

---

## Containerised Deployment

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY serve_fastapi.py .
COPY churn_pipeline.joblib .

EXPOSE 8080

CMD ["uvicorn", "serve_fastapi:app", "--host", "0.0.0.0", "--port", "8080"]
```

```bash
docker build -t churn-model:v1 .
docker run -p 8080:8080 churn-model:v1

# Test
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"age": 35, "income": 60000, "account_age_days": 365, "country": "UK", "plan_type": "premium"}'
```

---

## Monitoring and Data Drift

A model trained on historical data degrades when the real world changes. Monitoring catches this before it impacts business outcomes.

### Types of Drift

| Type | What Changes | Example |
|------|-------------|---------|
| **Data drift** (covariate shift) | Input feature distributions | Average income shifts due to inflation |
| **Concept drift** | Relationship between features and target | Customer behaviour changes after a product update |
| **Prediction drift** | Distribution of predictions | Model starts predicting "churn" for 50% of users instead of 10% |

### Detecting Drift with Evidently

```python
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset, TargetDriftPreset

# Compare training data distributions with production data
report = Report(metrics=[DataDriftPreset()])
report.run(reference_data=train_df, current_data=production_df)
report.save_html("drift_report.html")
```

### Simple Drift Detection

```python
from scipy.stats import ks_2samp

def detect_drift(reference, current, threshold=0.05):
    """Kolmogorov-Smirnov test for distribution shift."""
    results = {}
    for col in reference.columns:
        stat, p_value = ks_2samp(reference[col], current[col])
        results[col] = {
            "statistic": stat,
            "p_value": p_value,
            "drifted": p_value < threshold
        }
    return results

drift_results = detect_drift(X_train_df, X_production_df)
drifted_features = [k for k, v in drift_results.items() if v["drifted"]]
print(f"Drifted features: {drifted_features}")
```

### Monitoring Checklist

| What to Monitor | How | Alert When |
|-----------------|-----|-----------|
| Prediction distribution | Histogram comparison | Distribution shift (KS test, PSI) |
| Feature distributions | Per-feature KS test | p-value < 0.05 on key features |
| Model accuracy | Compare predictions with delayed ground truth | Accuracy drops below threshold |
| Latency | Request timing | P95 exceeds SLA |
| Data quality | Null rate, type mismatches, schema violations | Any unexpected nulls or types |
| Throughput | Requests per second | Sudden spikes or drops |

---

## Retraining Strategies

| Strategy | Trigger | Complexity |
|----------|---------|-----------|
| Scheduled | Fixed interval (weekly, monthly) | Low |
| Drift-triggered | Monitoring detects drift | Medium |
| Performance-triggered | Ground truth shows accuracy drop | Medium |
| Online learning | Continuous updates with each new data point | High |

---

## Key Takeaways

- MLflow is the standard for experiment tracking — log parameters, metrics, and models for every training run to enable comparison and reproducibility.
- Version everything: code (Git), data (DVC), models (MLflow registry), and environment (Docker) — you must be able to reproduce any past model.
- FastAPI is the recommended framework for production model serving — it provides type validation, async support, and auto-generated API docs.
- Containerise your serving application with Docker — this ensures the model runs identically in development, staging, and production.
- Data drift is inevitable — monitor input distributions, prediction distributions, and model accuracy using statistical tests or tools like Evidently.
- Establish a retraining strategy before deployment — at minimum, schedule periodic retraining and alert on significant drift.
