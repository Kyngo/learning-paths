---
title: "Unsupervised Learning"
weight: 6
---

# Unsupervised Learning

Unsupervised learning finds structure in data without labels. Clustering groups similar points together, dimensionality reduction compresses features while preserving structure, and anomaly detection identifies outliers. These techniques are essential for exploration, preprocessing, and problems where labels are expensive or unavailable.

---

## Clustering

### K-Means

K-Means partitions data into K clusters by iteratively assigning points to the nearest centroid and updating centroids.

**Algorithm:**
1. Initialise K centroids (k-means++ for smart initialisation)
2. Assign each point to the nearest centroid
3. Recompute centroids as the mean of assigned points
4. Repeat until convergence

```python
from sklearn.cluster import KMeans
from sklearn.datasets import make_blobs
import matplotlib.pyplot as plt
import numpy as np

X, y_true = make_blobs(n_samples=300, centers=4, cluster_std=0.8, random_state=42)

kmeans = KMeans(n_clusters=4, init="k-means++", n_init=10, random_state=42)
labels = kmeans.fit_predict(X)

plt.scatter(X[:, 0], X[:, 1], c=labels, cmap="viridis", s=20, alpha=0.7)
plt.scatter(kmeans.cluster_centers_[:, 0], kmeans.cluster_centers_[:, 1],
            c="red", marker="X", s=200, edgecolors="black")
plt.title("K-Means Clustering")
plt.show()
```

### Choosing K: The Elbow Method and Silhouette Score

```python
from sklearn.metrics import silhouette_score

inertias = []
silhouettes = []
K_range = range(2, 10)

for k in K_range:
    km = KMeans(n_clusters=k, n_init=10, random_state=42)
    km.fit(X)
    inertias.append(km.inertia_)
    silhouettes.append(silhouette_score(X, km.labels_))

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))
ax1.plot(K_range, inertias, "bo-")
ax1.set_xlabel("K")
ax1.set_ylabel("Inertia")
ax1.set_title("Elbow Method")

ax2.plot(K_range, silhouettes, "ro-")
ax2.set_xlabel("K")
ax2.set_ylabel("Silhouette Score")
ax2.set_title("Silhouette Analysis")
plt.tight_layout()
plt.show()
```

| Method | What It Measures | Choose K Where |
|--------|-----------------|---------------|
| Elbow | Within-cluster sum of squares (inertia) | Curve "bends" (diminishing returns) |
| Silhouette | Cohesion vs separation (-1 to 1) | Maximum score |
| Gap Statistic | Compares inertia to random reference | Largest gap |

### K-Means Limitations

- Assumes spherical clusters of similar size
- Sensitive to initialisation (mitigated by `n_init`)
- Requires K to be specified in advance
- Cannot detect non-convex clusters

### DBSCAN

Density-Based Spatial Clustering of Applications with Noise. Finds clusters of arbitrary shape and identifies noise points.

**Key parameters:**
- `eps` — maximum distance between two points in the same neighbourhood
- `min_samples` — minimum points to form a dense region (core point)

```python
from sklearn.cluster import DBSCAN
from sklearn.datasets import make_moons

X, _ = make_moons(n_samples=300, noise=0.05, random_state=42)

db = DBSCAN(eps=0.2, min_samples=5)
labels = db.fit_predict(X)

n_clusters = len(set(labels)) - (1 if -1 in labels else 0)
n_noise = (labels == -1).sum()

plt.scatter(X[:, 0], X[:, 1], c=labels, cmap="viridis", s=20)
plt.title(f"DBSCAN: {n_clusters} clusters, {n_noise} noise points")
plt.show()
```

| Advantage | Limitation |
|-----------|-----------|
| No need to specify K | Sensitive to eps and min_samples |
| Finds arbitrary-shape clusters | Struggles with varying densities |
| Identifies noise/outliers | Does not assign cluster labels to noise |
| Deterministic (no random init) | Slow on very large datasets (O(n²) without index) |

### Hierarchical Clustering

Builds a tree of clusters (dendrogram) by iteratively merging or splitting:

```python
from sklearn.cluster import AgglomerativeClustering
from scipy.cluster.hierarchy import dendrogram, linkage

# Dendrogram for visual inspection
linked = linkage(X, method="ward")
plt.figure(figsize=(10, 5))
dendrogram(linked, truncate_mode="lastp", p=20)
plt.title("Hierarchical Clustering Dendrogram")
plt.xlabel("Cluster Size")
plt.ylabel("Distance")
plt.show()

# Agglomerative clustering
agg = AgglomerativeClustering(n_clusters=4, linkage="ward")
labels = agg.fit_predict(X)
```

| Linkage | How It Measures Distance | Behaviour |
|---------|------------------------|-----------|
| Ward | Minimises within-cluster variance | Compact, equal-sized clusters |
| Complete | Maximum distance between points | Compact clusters |
| Average | Mean distance between all pairs | Compromise |
| Single | Minimum distance between points | Chain-like clusters |

---

## Dimensionality Reduction

### PCA (Principal Component Analysis)

PCA finds the directions of maximum variance and projects data onto them. It is a linear transformation.

```python
from sklearn.decomposition import PCA
from sklearn.datasets import load_digits

X, y = load_digits(return_X_y=True)  # 64 features (8x8 pixel images)

pca = PCA(n_components=2)
X_2d = pca.fit_transform(X)

plt.scatter(X_2d[:, 0], X_2d[:, 1], c=y, cmap="tab10", s=5, alpha=0.7)
plt.colorbar(label="Digit")
plt.xlabel(f"PC1 ({pca.explained_variance_ratio_[0]:.1%})")
plt.ylabel(f"PC2 ({pca.explained_variance_ratio_[1]:.1%})")
plt.title("PCA — Digits Dataset")
plt.show()
```

### Choosing the Number of Components

```python
pca_full = PCA().fit(X)
cumvar = np.cumsum(pca_full.explained_variance_ratio_)

plt.plot(cumvar)
plt.xlabel("Number of Components")
plt.ylabel("Cumulative Explained Variance")
plt.axhline(y=0.95, color="r", linestyle="--", label="95% variance")
plt.legend()
plt.title("PCA Explained Variance")
plt.show()

# Keep 95% of variance
n_components_95 = np.argmax(cumvar >= 0.95) + 1
print(f"Components for 95% variance: {n_components_95}")
```

### PCA Use Cases

| Use Case | Benefit |
|----------|---------|
| Visualisation | Reduce to 2D/3D for plotting |
| Noise reduction | Remove low-variance components |
| Feature compression | Reduce dimensionality before training |
| Multicollinearity | Orthogonal components eliminate correlation |

### t-SNE

t-Distributed Stochastic Neighbour Embedding. A non-linear method designed for **visualisation** (typically 2D or 3D). Preserves local structure but not global distances.

```python
from sklearn.manifold import TSNE

tsne = TSNE(n_components=2, perplexity=30, random_state=42)
X_tsne = tsne.fit_transform(X)

plt.scatter(X_tsne[:, 0], X_tsne[:, 1], c=y, cmap="tab10", s=5, alpha=0.7)
plt.colorbar(label="Digit")
plt.title("t-SNE — Digits Dataset")
plt.show()
```

**Key parameter:** `perplexity` (5–50) controls the balance between local and global structure. Try multiple values.

**Limitations:** Slow on large datasets, non-deterministic, distances between clusters are not meaningful, cannot transform new data (no `transform()` method).

### UMAP

Uniform Manifold Approximation and Projection. Faster than t-SNE, preserves more global structure, and supports `transform()` for new data.

```python
from umap import UMAP

reducer = UMAP(n_components=2, n_neighbors=15, min_dist=0.1, random_state=42)
X_umap = reducer.fit_transform(X)

plt.scatter(X_umap[:, 0], X_umap[:, 1], c=y, cmap="tab10", s=5, alpha=0.7)
plt.colorbar(label="Digit")
plt.title("UMAP — Digits Dataset")
plt.show()
```

### Comparison

| Method | Linear? | Speed | Global Structure | Transform New Data | Primary Use |
|--------|---------|-------|-----------------|-------------------|-------------|
| PCA | Yes | Fast | ✅ Preserved | ✅ Yes | Compression, preprocessing |
| t-SNE | No | Slow | ❌ Not preserved | ❌ No | Visualisation only |
| UMAP | No | Fast | ⚠️ Partially | ✅ Yes | Visualisation, preprocessing |

---

## Anomaly Detection

### Isolation Forest

Isolates anomalies by randomly selecting features and split values. Anomalies are easier to isolate (shorter path in the tree) than normal points.

```python
from sklearn.ensemble import IsolationForest

# Generate data with outliers
np.random.seed(42)
X_normal = np.random.randn(300, 2)
X_outliers = np.random.uniform(-4, 4, size=(20, 2))
X_all = np.vstack([X_normal, X_outliers])

iso = IsolationForest(n_estimators=100, contamination=0.05, random_state=42)
predictions = iso.fit_predict(X_all)  # 1 = normal, -1 = anomaly

plt.scatter(X_all[predictions == 1, 0], X_all[predictions == 1, 1],
            c="blue", s=20, label="Normal", alpha=0.5)
plt.scatter(X_all[predictions == -1, 0], X_all[predictions == -1, 1],
            c="red", s=40, label="Anomaly", marker="x")
plt.legend()
plt.title("Isolation Forest")
plt.show()
```

| Parameter | Effect |
|-----------|--------|
| `n_estimators` | Number of isolation trees (100 is usually sufficient) |
| `contamination` | Expected proportion of outliers in the dataset |
| `max_samples` | Number of samples to draw for each tree |

### Other Anomaly Detection Methods

| Method | Approach | Best For |
|--------|----------|----------|
| Isolation Forest | Random partitioning, anomalies have short paths | General-purpose, high-dimensional |
| Local Outlier Factor | Density-based, compares local density to neighbours | Varying-density data |
| One-Class SVM | Learns boundary around normal data | Small datasets, well-defined normal class |
| Elliptic Envelope | Fits Gaussian to data, flags low-probability points | Normally distributed features |

---

## Key Takeaways

- K-Means is simple and fast but assumes spherical clusters and requires K in advance — use the elbow method or silhouette score to choose K.
- DBSCAN finds arbitrary-shape clusters and identifies noise, but is sensitive to the `eps` parameter — it excels where K-Means fails.
- PCA is the go-to for dimensionality reduction: it is fast, linear, and provides explained variance ratios for choosing the number of components.
- t-SNE and UMAP are for visualisation — t-SNE is slower and cannot transform new data; UMAP is faster and preserves more global structure.
- Isolation Forest is the default anomaly detection method: it is fast, handles high dimensions well, and requires minimal tuning beyond the contamination parameter.
- Unsupervised methods are essential for exploration, preprocessing, and feature engineering — they complement supervised learning rather than replacing it.
