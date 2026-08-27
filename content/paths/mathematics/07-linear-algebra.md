---
title: "Linear Algebra"
weight: 7
---

# Linear Algebra

Linear algebra is the mathematics of vectors, matrices, and linear transformations. It is the computational engine behind machine learning, computer graphics, data compression, search engines, and recommendation systems.

---

## Vectors

A **vector** is an ordered list of numbers. It can represent a point in space, a direction, a data record, or a feature embedding.

```
v = [3, 4]           2D vector
w = [1, 0, -2, 5]    4D vector
```

### Vector Operations

| Operation | Formula | Example |
|-----------|---------|---------|
| Addition | u + v = [u₁+v₁, u₂+v₂, ...] | [1,2] + [3,4] = [4,6] |
| Scalar multiplication | c·v = [cv₁, cv₂, ...] | 3·[1,2] = [3,6] |
| Dot product | u·v = Σ uᵢvᵢ | [1,2]·[3,4] = 3+8 = 11 |
| Magnitude (norm) | ‖v‖ = √(Σ vᵢ²) | ‖[3,4]‖ = √(9+16) = 5 |
| Unit vector | v̂ = v / ‖v‖ | [3,4]/5 = [0.6, 0.8] |

### The Dot Product

The dot product captures the notion of **alignment** between two vectors:

```
u · v = ‖u‖ × ‖v‖ × cos(θ)
```

Where θ is the angle between u and v.

| Dot product | Meaning |
|------------|---------|
| > 0 | Vectors point in similar directions (θ < 90°) |
| = 0 | Vectors are perpendicular (orthogonal) |
| < 0 | Vectors point in opposite directions (θ > 90°) |

### Cosine Similarity

Normalise the dot product by both magnitudes to get a similarity score independent of vector length:

```
cos_sim(u, v) = (u · v) / (‖u‖ × ‖v‖)
```

Range: [-1, 1]. Used everywhere in ML:

| Application | What Vectors Represent |
|-------------|----------------------|
| Search / RAG | Document and query embeddings |
| Recommendations | User and item embeddings |
| NLP | Word or sentence embeddings |
| Image retrieval | CNN feature vectors |
| Anomaly detection | Normal vs. observed behaviour vectors |

---

## Matrices

A **matrix** is a rectangular array of numbers. An m × n matrix has m rows and n columns.

```
A = ┌ 1  2  3 ┐
    │ 4  5  6 │    (2 × 3 matrix)
    └ 7  8  9 ┘

    (this is actually 3×3, let me be precise)

A = ┌ 1  2 ┐
    └ 3  4 ┘       (2 × 2 matrix)
```

### Matrix Operations

| Operation | Rule | Dimension Result |
|-----------|------|-----------------|
| Addition | A + B (element-wise) | Same dimensions required |
| Scalar multiplication | cA (multiply each element) | Same dimensions |
| Matrix multiplication | C = AB where Cᵢⱼ = Σ Aᵢₖ Bₖⱼ | (m×n) × (n×p) = (m×p) |
| Transpose | (Aᵀ)ᵢⱼ = Aⱼᵢ (swap rows/columns) | (m×n) → (n×m) |

### Matrix Multiplication

The entry in row i, column j of AB is the **dot product** of row i of A with column j of B.

```
┌ 1  2 ┐   ┌ 5  6 ┐   ┌ 1×5+2×7  1×6+2×8 ┐   ┌ 19  22 ┐
└ 3  4 ┘ × └ 7  8 ┘ = └ 3×5+4×7  3×6+4×8 ┘ = └ 43  50 ┘
```

**Key properties:**
- Not commutative: AB ≠ BA in general
- Associative: (AB)C = A(BC)
- Distributive: A(B + C) = AB + AC

**Computational cost:** Multiplying an (m×n) matrix by an (n×p) matrix requires O(mnp) operations.

### Special Matrices

| Matrix | Property | Example |
|--------|----------|---------|
| Identity (I) | AI = IA = A | Diagonal of 1s |
| Diagonal | Nonzero only on main diagonal | Scaling transform |
| Symmetric | A = Aᵀ | Covariance matrices |
| Orthogonal | AᵀA = I, A⁻¹ = Aᵀ | Rotation matrices |
| Sparse | Most entries are zero | Adjacency matrices of real graphs |
| Triangular | All entries above (or below) diagonal are zero | LU decomposition result |

---

## Systems of Linear Equations

A system of m equations in n unknowns can be written as:

```
Ax = b
```

Where A is the coefficient matrix, x is the unknowns vector, and b is the constants vector.

### Solving Methods

| Method | Time | Best For |
|--------|------|----------|
| Gaussian elimination | O(n³) | General dense systems |
| LU decomposition | O(n³) | Solving with multiple b vectors |
| Iterative methods (Jacobi, Gauss-Seidel) | Varies | Large sparse systems |
| Matrix inverse (x = A⁻¹b) | O(n³) | Theoretical; numerically avoid |

### When Solutions Exist

| Condition | Solutions |
|-----------|----------|
| det(A) ≠ 0 (full rank) | Exactly one solution |
| Rank(A) < n, consistent | Infinitely many solutions |
| Inconsistent | No solution |

---

## Determinants

The determinant of a square matrix is a scalar that captures how the matrix scales areas/volumes.

### 2×2 Determinant

```
det ┌ a  b ┐ = ad - bc
    └ c  d ┘
```

### Properties

| Property | Meaning |
|----------|---------|
| det(A) = 0 | Matrix is singular (not invertible) |
| det(A) ≠ 0 | Matrix is invertible |
| det(AB) = det(A) × det(B) | Determinants multiply |
| det(Aᵀ) = det(A) | Transpose preserves determinant |
| \|det(A)\| | Scaling factor for area/volume |

### Geometric Interpretation

The absolute value of the determinant gives the factor by which the transformation scales area (2D) or volume (3D):

- det = 1: preserves area (rotation)
- det = 2: doubles area
- det = 0: collapses to lower dimension (squishes to a line or point)
- det < 0: flips orientation (mirror)

---

## Eigenvalues and Eigenvectors

An **eigenvector** of a matrix A is a nonzero vector v that, when transformed by A, only changes in scale:

```
Av = λv
```

Where λ (lambda) is the **eigenvalue** — the scaling factor.

### Finding Eigenvalues

Solve the **characteristic equation:**

```
det(A - λI) = 0
```

### Example

```
A = ┌ 2  1 ┐
    └ 1  2 ┘

det(A - λI) = det ┌ 2-λ   1  ┐ = (2-λ)² - 1 = λ² - 4λ + 3 = (λ-1)(λ-3)
                   └  1   2-λ ┘

Eigenvalues: λ₁ = 1, λ₂ = 3
```

### Why Eigenvalues Matter

| Application | Role of Eigenvalues/Eigenvectors |
|-------------|--------------------------------|
| PCA (dimensionality reduction) | Eigenvectors of covariance matrix are principal components; eigenvalues are variances |
| Google PageRank | The dominant eigenvector of the link matrix gives page importance |
| Stability analysis | System is stable if all eigenvalues have \|λ\| < 1 |
| Vibration / resonance | Eigenvalues are natural frequencies |
| Graph spectral analysis | Eigenvalues of adjacency/Laplacian reveal graph structure |
| Markov chains | Stationary distribution is the eigenvector for λ = 1 |

---

## Linear Transformations

A matrix can be interpreted as a **transformation** that maps vectors from one space to another.

### Common 2D Transformations

| Transformation | Matrix | Effect |
|---------------|--------|--------|
| Scaling | [[sₓ, 0], [0, s_y]] | Scale by sₓ horizontal, s_y vertical |
| Rotation by θ | [[cos θ, -sin θ], [sin θ, cos θ]] | Rotate counter-clockwise |
| Reflection (x-axis) | [[1, 0], [0, -1]] | Flip vertically |
| Shear (horizontal) | [[1, k], [0, 1]] | Slant horizontally |
| Projection onto x-axis | [[1, 0], [0, 0]] | Collapse y-component |

### Composition of Transformations

Applying transformation B then A is the matrix product AB:

```
(AB)v = A(Bv)
```

The order matters (matrix multiplication is not commutative). In graphics pipelines, transformations are composed right-to-left: model → world → view → projection.

---

## Decompositions

Matrix decompositions break a matrix into simpler components for efficient computation.

### Singular Value Decomposition (SVD)

Any m × n matrix A can be decomposed as:

```
A = UΣVᵀ
```

Where:
- U is m × m orthogonal (left singular vectors)
- Σ is m × n diagonal (singular values, σ₁ ≥ σ₂ ≥ ... ≥ 0)
- V is n × n orthogonal (right singular vectors)

**Applications:**

| Application | How SVD Is Used |
|-------------|----------------|
| Image compression | Keep top-k singular values, discard the rest |
| Latent semantic analysis | Reduce term-document matrix to key concepts |
| Recommender systems | Matrix factorisation (Netflix Prize) |
| Pseudoinverse | Solve least-squares problems |
| Noise reduction | Small singular values often correspond to noise |

### Truncated SVD

Keep only the top k singular values. The resulting rank-k matrix is the **best rank-k approximation** (minimises Frobenius norm of the error):

```
A ≈ U_k Σ_k V_k^T
```

An image stored as a 1000×1000 matrix (1M values) with rank-50 SVD uses only 50 × (1000 + 1000 + 1) ≈ 100K values — 10× compression.

---

## Vector Spaces (Brief)

A **vector space** is a collection of vectors closed under addition and scalar multiplication. The essential concepts:

| Concept | Meaning |
|---------|---------|
| Basis | Minimal set of vectors that span the space |
| Dimension | Number of vectors in a basis |
| Span | All vectors reachable by linear combinations of a set |
| Linear independence | No vector in the set can be written as a combination of the others |
| Rank of a matrix | Dimension of the column space (number of linearly independent columns) |
| Null space | Set of vectors x where Ax = 0 |

**Rank-nullity theorem:** For an m × n matrix A:

```
rank(A) + nullity(A) = n
```

---

## Key Takeaways

- Vectors represent data points, directions, and embeddings. The dot product measures alignment; cosine similarity measures it normalised.
- Matrix multiplication is the workhorse of ML, graphics, and scientific computing. It composes linear transformations.
- Eigenvalues and eigenvectors reveal the "natural axes" of a transformation — they are the key to PCA, PageRank, and stability analysis.
- SVD is the Swiss Army knife of matrix decompositions: compression, noise reduction, recommendations, and pseudoinverses.
- Linear algebra is not abstract — every time you query a vector database, rotate a 3D model, train a neural network, or run PageRank, you are doing linear algebra.
