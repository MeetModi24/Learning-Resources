# 23 — Clustering

**Playlist videos 128–132.** This section moves from supervised learning (labels available) into
**unsupervised learning** (Section 01, §3): the data has no `y`, only `X`, and the goal is to find
structure the data already has — specifically, to **group similar points together**. Three
algorithms, three different notions of "similar": K-Means (distance to a center), Agglomerative
Hierarchical Clustering (distance between groups, bottom-up), and DBSCAN (density).

Common use cases: **customer segmentation** (group users by behavior for targeted marketing),
**anomaly detection** (points that don't fit any cluster are suspicious), document/image grouping,
and as a preprocessing step before supervised modeling.

---

## 1. K-Means Clustering — Geometric Intuition

**Goal:** partition `n` points into `K` clusters, where `K` is chosen up front, such that points in
the same cluster are close together and points in different clusters are far apart.

### The algorithm (Lloyd's algorithm)

1. **Choose K** — the number of clusters (a hyperparameter you must decide).
2. **Initialize K centroids** — pick K points (randomly, or via a smarter scheme, see below) as the
   starting cluster centers.
3. **Assign step** — assign every data point to its **nearest centroid** (typically by Euclidean
   distance). This partitions the data into K clusters.
4. **Update step** — recompute each centroid as the **mean** of all points currently assigned to it.
5. **Repeat** steps 3–4 until the assignments stop changing (or centroids move below some
   tolerance) — the algorithm has **converged**.

Each iteration can only decrease (or keep equal) the objective, so it's guaranteed to converge, but
not necessarily to the global optimum (see initialization, below).

### The objective: inertia (within-cluster sum of squares)

K-Means implicitly minimizes:

```
Inertia = Σ_clusters Σ_(points i in cluster) ||x_i - centroid||²
```

i.e., the total squared distance from each point to its own cluster's centroid. Lower inertia means
tighter clusters. This is also called **WCSS** (within-cluster sum of squares). K-Means doesn't
optimize this globally — it's a greedy, iterative heuristic — but it's the quantity that decreases
every iteration and the one you use to pick K (below).

### Feature scaling is required

Distance-based, exactly like KNN and SVM (Sections 15–16): if one feature has a much larger scale
than others, it dominates the Euclidean distance and the clustering is really just clustering on
that one feature. Always scale (standardize) features before K-Means.

### Sensitivity to initialization — k-means++

Random centroid initialization can land in a bad local optimum — different random seeds give
different final clusters, some clearly worse than others (e.g., two centroids initialized inside
the same true cluster, splitting it, while two real clusters get merged under one centroid).

**k-means++** fixes this with a smarter initialization: pick the first centroid randomly, then pick
each subsequent centroid with probability proportional to its squared distance from the nearest
already-chosen centroid. This spreads the initial centroids out across the data, making it far more
likely they land near the true cluster centers. It's the scikit-learn default (`init='k-means++'`).

### Choosing K

K-Means needs K decided in advance, so you need a way to choose it.

- **Elbow method:** run K-Means for a range of K values, plot **inertia vs K**. Inertia always
  decreases as K increases (more clusters = tighter fit), but the *rate* of decrease drops sharply
  after the "right" K and flattens out afterward — that bend is the "elbow." Pick K at the elbow.
  Downside: the elbow is often ambiguous/subjective on real data.
- **Silhouette score:** for each point, compare (a) its average distance to points in its own
  cluster (cohesion) against (b) its average distance to points in the nearest *other* cluster
  (separation). The score is `(b - a) / max(a, b)`, ranging from -1 to 1 — near +1 means the point
  is well-matched to its own cluster and far from others; near 0 means it's on a cluster boundary;
  negative means it's probably in the wrong cluster. Average this over all points for each K and
  pick the K that maximizes the mean silhouette score. More quantitative/objective than the elbow.

### Assumptions and limitations

- Assumes clusters are roughly **spherical (globular)** and **similar in size/density**, because
  it's fundamentally comparing distances to a single center per cluster.
- Struggles with **non-globular shapes** (crescents, rings, elongated clusters) — it will cut
  through them incorrectly.
- Sensitive to **outliers**, since the mean (used to update centroids) is itself outlier-sensitive.
- **K must be specified in advance** — you don't get it "for free" from the data (contrast with
  hierarchical clustering and DBSCAN below).

---

## 2. K-Means Clustering in Python — Practical Example

Minimal usage with scikit-learn:

```python
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans

X_scaled = StandardScaler().fit_transform(X)          # scaling is required

kmeans = KMeans(n_clusters=4, init='k-means++', n_init=10, random_state=42)
labels = kmeans.fit_predict(X_scaled)                  # cluster assignment per point

kmeans.inertia_        # WCSS for this K — use across a range of K for the elbow plot
kmeans.cluster_centers_  # the K centroid coordinates
```

`n_init` runs the whole algorithm multiple times with different centroid seeds and keeps the best
(lowest-inertia) run — a practical hedge against the initialization sensitivity above, on top of
k-means++.

To pick K, sweep it and plot inertia (elbow) and/or the silhouette score:

```python
from sklearn.metrics import silhouette_score

for k in range(2, 11):
    km = KMeans(n_clusters=k, init='k-means++', n_init=10, random_state=42).fit(X_scaled)
    print(k, km.inertia_, silhouette_score(X_scaled, km.labels_))
```

---

## 3. K-Means Clustering From Scratch

Implementing Lloyd's algorithm directly makes the mechanics concrete — there's no hidden math
beyond "distance" and "mean":

```python
import numpy as np

def kmeans(X, k, n_iter=100, seed=42):
    rng = np.random.default_rng(seed)
    centroids = X[rng.choice(len(X), k, replace=False)]   # naive random init (not k-means++)

    for _ in range(n_iter):
        # assign step: nearest centroid by Euclidean distance
        dists = np.linalg.norm(X[:, None, :] - centroids[None, :, :], axis=2)
        labels = dists.argmin(axis=1)

        # update step: recompute centroids as cluster means
        new_centroids = np.array([X[labels == j].mean(axis=0) for j in range(k)])

        if np.allclose(centroids, new_centroids):          # converged
            break
        centroids = new_centroids

    return labels, centroids
```

The two moving pieces map exactly to the algorithm above: `dists.argmin(axis=1)` is the assign step
(nearest centroid), and the per-cluster `.mean(axis=0)` is the update step. Convergence is checked
by comparing centroids across iterations — when they stop moving, the labels are stable too.

---

## 4. Agglomerative Hierarchical Clustering

A completely different strategy: **bottom-up (agglomerative) merging**, with no need to fix K
up front.

### The algorithm

1. Start with **every point as its own cluster** (n points → n clusters).
2. Find the **two closest clusters** and **merge** them into one.
3. Repeat step 2 — recomputing distances between the now-fewer clusters — until only **one cluster**
   remains (containing everything).

Each merge is recorded, building a **dendrogram**: a tree showing, at each height, which
clusters merged and at what distance. To get an actual clustering with a specific number of
clusters, you **cut the dendrogram** at a chosen height — every branch below the cut becomes one
cluster. Cutting higher up gives fewer, larger clusters; cutting lower gives more, smaller ones.
This is how you effectively "read K off the data" after the fact, instead of committing to it
before running the algorithm.

### Linkage criteria — how do you measure distance *between clusters*?

Once a cluster has more than one point, "distance between two clusters" needs a rule (a point-to-point distance rule, like Euclidean, is still used underneath):

- **Single linkage** — distance between the two **closest** points, one from each cluster.
  Tends to produce elongated, "chained" clusters; sensitive to noise bridging two true clusters.
- **Complete linkage** — distance between the two **farthest** points, one from each cluster.
  Tends to produce compact, evenly-sized clusters; less prone to chaining.
- **Average linkage** — the **average** distance across all pairs of points between the two
  clusters. A middle ground between single and complete.
- **Ward linkage** — merge the pair of clusters that produces the **smallest increase in
  within-cluster variance** (similar spirit to K-Means' inertia). Tends to produce compact,
  similarly-sized clusters and is the most commonly used default.

### Pros and cons

- **Pros:** no need to pick K upfront (read it off the dendrogram); the dendrogram itself is a
  useful visualization of nested structure/similarity in the data, not just a final partition.
- **Cons:** merges are **greedy and permanent** — once two clusters are merged there's no going
  back to reconsider, even if a later merge reveals it was wrong. Computing and updating all
  pairwise cluster distances is **O(n²) or worse** in time and memory, so it does not scale to
  large datasets the way K-Means or DBSCAN do.

### Minimal sklearn usage

```python
from sklearn.cluster import AgglomerativeClustering
from scipy.cluster.hierarchy import dendrogram, linkage
import matplotlib.pyplot as plt

# visualize the dendrogram to pick a cut height
Z = linkage(X_scaled, method='ward')
dendrogram(Z)
plt.show()

# cut to a specific number of clusters
agg = AgglomerativeClustering(n_clusters=4, linkage='ward')
labels = agg.fit_predict(X_scaled)
```

---

## 5. DBSCAN — Density-Based Clustering

A third strategy: define clusters as **dense regions of points, separated by sparse regions**,
rather than by distance to a center (K-Means) or nested merges (hierarchical). This is the key
conceptual shift — DBSCAN doesn't assume any particular cluster *shape* at all.

### Parameters

- **`eps`** — the radius defining a point's neighborhood.
- **`min_samples`** — the minimum number of points (including itself) that must fall within `eps`
  of a point for that neighborhood to count as **dense**.

### Point classification

- **Core point** — has at least `min_samples` points within `eps` of it (including itself). Dense
  neighborhood on its own.
- **Border point** — not a core point itself, but falls within `eps` of a core point. It's part of
  a cluster, but on its edge.
- **Noise / outlier point** — neither a core point nor within `eps` of any core point. Not assigned
  to any cluster.

A cluster is formed by taking a core point and all points **density-reachable** from it (chaining
through neighboring core points), plus any border points attached along the way.

### Strengths

- Finds clusters of **arbitrary shape** (crescents, rings, elongated blobs) — no globularity
  assumption, unlike K-Means.
- **Doesn't require K** to be specified — the number of clusters falls out of the density
  structure.
- **Robust to outliers** — they're explicitly labeled as noise rather than being forced into the
  nearest cluster (which is what K-Means and hierarchical clustering both do).

### Weaknesses

- Struggles when clusters have **varying density** — a single `eps`/`min_samples` pair can't
  simultaneously suit a tight dense cluster and a loose sparse one; the sparse one may get
  shredded into noise or the tight one may get merged with neighbors.
- Degrades in **high dimensions** — distance becomes less meaningful as dimensionality grows (the
  same curse-of-dimensionality issue as KNN, Section 08), making "dense neighborhood" harder to
  define.
- **Sensitive to `eps`/`min_samples`** — no automatic way to choose them; usually tuned by
  inspecting a k-distance plot or trial and error.

### Minimal sklearn usage

```python
from sklearn.cluster import DBSCAN

db = DBSCAN(eps=0.5, min_samples=5)
labels = db.fit_predict(X_scaled)     # noise points are labeled -1
```

---

## Choosing between the three

| | K-Means | Agglomerative | DBSCAN |
|---|---|---|---|
| Must specify K upfront? | Yes | No (cut dendrogram after) | No |
| Cluster shape assumption | Spherical, similar size | Depends on linkage; generally compact-ish | Arbitrary shape |
| Handles outliers | Poorly (pulled into nearest cluster) | Poorly (always merged in eventually) | Well (labeled as noise) |
| Scales to large n | Yes (fast, roughly linear per iteration) | No (O(n²) or worse) | Reasonably (with spatial indexing) |
| Varying cluster density | Assumes similar density | No explicit density notion | Struggles |
| Extra output | Just labels | Dendrogram (nested structure) | Core/border/noise typing |

Rough rule of thumb: reach for **K-Means** when clusters are roughly round and similarly sized and
you're willing to pick/tune K (elbow/silhouette); reach for **Agglomerative** when you want to see
the nested/hierarchical structure or don't want to commit to K in advance, on a dataset small
enough for O(n²) to be fine; reach for **DBSCAN** when clusters have irregular shapes and/or the
data has real outliers/noise you want identified rather than forced into a cluster.

---

## Key takeaways

- Clustering is **unsupervised**: no `y`, just `X` — the algorithm finds groups of similar points
  on its own (Section 01, §3). Used for segmentation, anomaly detection, and exploratory structure.
- **K-Means**: choose K, initialize centroids (k-means++ to avoid bad local optima), iterate
  assign-nearest-centroid → recompute-centroid-as-mean until stable. Minimizes inertia (WCSS).
  Requires scaling; needs K chosen via elbow or silhouette; assumes spherical, similar-size
  clusters; sensitive to outliers.
- **Agglomerative Hierarchical Clustering**: start with every point as its own cluster, repeatedly
  merge the two closest (by single/complete/average/ward linkage) into a dendrogram, then cut at a
  height to get K clusters after the fact. No K needed upfront, but greedy/irreversible merges and
  O(n²)+ cost limit it to smaller datasets.
- **DBSCAN**: clusters are dense regions (core points and their reachable border points) separated
  by sparse regions (noise). Governed by `eps` and `min_samples`. Finds arbitrary shapes, needs no
  K, and naturally flags outliers — but struggles with varying density and high dimensions.
- No single algorithm dominates: shape assumptions, outlier handling, and scalability trade off
  differently across the three (see comparison table).

**Next:** [24 — Imbalanced Data](24-imbalanced-data.md) — handling skewed class distributions in
classification.
