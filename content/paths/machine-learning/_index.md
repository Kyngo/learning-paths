---
title: "Machine Learning"
weight: 47
bookCollapseSection: true
---

# Machine Learning

Classical machine learning — from linear regression to production pipelines. The mathematics, algorithms, and engineering practices that underpin every ML system, before deep learning enters the picture.

## Overview

Machine learning is the discipline of building systems that learn patterns from data rather than being explicitly programmed. This path covers **classical (non-deep-learning) ML**: the algorithms, evaluation techniques, and engineering workflows that remain the backbone of most production ML systems today.

Despite the hype around deep learning, the majority of real-world ML in production — fraud detection, recommendation engines, pricing models, churn prediction, demand forecasting — runs on gradient-boosted trees, logistic regression, and well-engineered feature pipelines. Understanding these foundations is not optional.

This path uses **Python** throughout, with **scikit-learn** as the primary framework, supplemented by pandas, NumPy, and matplotlib. Every concept is paired with working code.

## Prerequisites

- Comfortable with Python (functions, classes, list comprehensions)
- Basic familiarity with NumPy and pandas
- High-school level statistics (mean, standard deviation, probability)
- Linear algebra basics helpful but not required (covered where needed)

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Introduction to Machine Learning]({{< relref "01-introduction" >}}) | Supervised, unsupervised, reinforcement learning, bias-variance tradeoff, overfitting, ML workflow |
| 2 | [Data Preparation]({{< relref "02-data-preparation" >}}) | Feature engineering, missing data, categorical encoding, scaling, train/validation/test splits |
| 3 | [Linear Models]({{< relref "03-linear-models" >}}) | Linear regression, logistic regression, gradient descent, regularisation (L1/L2/ElasticNet) |
| 4 | [Decision Trees & Ensembles]({{< relref "04-decision-trees-ensembles" >}}) | Decision trees, random forests, gradient boosting, XGBoost, LightGBM, CatBoost |
| 5 | [Support Vector Machines]({{< relref "05-support-vector-machines" >}}) | Linear SVM, kernel trick, RBF/polynomial kernels, soft margin, SVR |
| 6 | [Unsupervised Learning]({{< relref "06-unsupervised-learning" >}}) | K-means, DBSCAN, hierarchical clustering, PCA, t-SNE, UMAP, anomaly detection |
| 7 | [Model Evaluation]({{< relref "07-model-evaluation" >}}) | Accuracy, precision, recall, F1, ROC-AUC, confusion matrix, cross-validation, learning curves |
| 8 | [Hyperparameter Tuning]({{< relref "08-hyperparameter-tuning" >}}) | Grid search, random search, Bayesian optimisation, Optuna, early stopping |
| 9 | [Feature Selection & Engineering]({{< relref "09-feature-selection-engineering" >}}) | Filter/wrapper/embedded methods, mutual information, permutation importance, target encoding |
| 10 | [Time Series]({{< relref "10-time-series" >}}) | Stationarity, ARIMA, seasonal decomposition, lag features, Prophet, time series CV |
| 11 | [ML Pipelines & scikit-learn]({{< relref "11-ml-pipelines" >}}) | Pipeline, ColumnTransformer, custom transformers, serialisation, reproducibility |
| 12 | [MLOps & Deployment]({{< relref "12-mlops-deployment" >}}) | MLflow, experiment tracking, model versioning, serving, monitoring, data drift |

## Algorithm Quick Reference

| Algorithm | Type | Best For | Interpretable? |
|-----------|------|----------|----------------|
| Linear Regression | Supervised (regression) | Continuous targets, linear relationships | ✅ High |
| Logistic Regression | Supervised (classification) | Binary/multiclass, baseline models | ✅ High |
| Decision Tree | Supervised (both) | Explainable models, feature discovery | ✅ High |
| Random Forest | Supervised (both) | General-purpose, robust to noise | ⚠️ Medium |
| Gradient Boosting | Supervised (both) | Tabular data competitions, high accuracy | ⚠️ Medium |
| SVM | Supervised (both) | High-dimensional data, small datasets | ❌ Low |
| K-Means | Unsupervised (clustering) | Customer segmentation, data exploration | ✅ High |
| DBSCAN | Unsupervised (clustering) | Arbitrary-shape clusters, outlier detection | ⚠️ Medium |
| PCA | Unsupervised (dim. reduction) | Visualisation, noise reduction, compression | ⚠️ Medium |
| Isolation Forest | Unsupervised (anomaly) | Fraud detection, outlier identification | ⚠️ Medium |

## Python Environment Setup

```bash
# Create a virtual environment
python -m venv ml-env
source ml-env/bin/activate

# Install core dependencies
pip install scikit-learn pandas numpy matplotlib seaborn jupyter

# Optional but recommended
pip install xgboost lightgbm catboost optuna mlflow
```

## Recommended Learning Order

Follow the sections in order — each builds on the previous:

```text
Introduction → Data Preparation → Linear Models → Trees & Ensembles
→ SVM → Unsupervised → Evaluation → Tuning → Feature Engineering
→ Time Series → Pipelines → MLOps
```

Sections 7–9 (Evaluation, Tuning, Feature Engineering) are cross-cutting and apply to every algorithm covered earlier.
