---
title: "Graph Theory"
weight: 6
---

# Graph Theory

Graphs model relationships between things — networks, dependencies, routes, social connections, state machines. Graph theory provides the language and algorithms for navigating, searching, and optimising these structures.

---

## Definitions

A **graph** G = (V, E) consists of:
- **V** — a set of **vertices** (nodes)
- **E** — a set of **edges** (connections between vertices)

### Types of Graphs

| Type | Description | Example |
|------|------------|---------|
| Undirected | Edges have no direction | Facebook friendships |
| Directed (digraph) | Edges have direction (u → v) | Twitter follows, dependencies |
| Weighted | Edges have numeric weights | Road distances, latencies |
| Unweighted | All edges are equal | Simple connectivity |
| Simple | No self-loops or parallel edges | Most theoretical graphs |
| Multigraph | Allows parallel edges | Flight routes between cities |
| Complete (Kₙ) | Edge between every pair of vertices | Fully connected network |
| Bipartite | Vertices split into two sets, edges only between sets | Job assignments, matching |
| DAG | Directed acyclic graph | Task dependencies, Git commits |

### Terminology

| Term | Definition |
|------|-----------|
| Adjacent | Two vertices connected by an edge |
| Degree | Number of edges incident to a vertex |
| In-degree / Out-degree | Edges coming in / going out (directed graphs) |
| Path | Sequence of vertices where each consecutive pair is connected |
| Cycle | Path that starts and ends at the same vertex |
| Connected | Every vertex is reachable from every other (undirected) |
| Strongly connected | Every vertex reachable from every other following edge directions (directed) |
| Component | Maximal connected subgraph |

### The Handshaking Lemma

In any undirected graph, the sum of all vertex degrees equals twice the number of edges:

```
Σ deg(v) = 2|E|
```

**Corollary:** The number of vertices with odd degree is always even.

---

## Representations

### Adjacency Matrix

A |V| × |V| matrix where entry (i, j) = 1 if there is an edge from i to j.

```
     A B C D
A  [ 0 1 1 0 ]
B  [ 1 0 1 1 ]
C  [ 1 1 0 0 ]
D  [ 0 1 0 0 ]
```

- Space: O(V²)
- Check edge exists: O(1)
- List all neighbours: O(V)
- Best for: dense graphs, matrix operations (PageRank, spectral analysis)

### Adjacency List

Each vertex stores a list of its neighbours.

```
A → [B, C]
B → [A, C, D]
C → [A, B]
D → [B]
```

- Space: O(V + E)
- Check edge exists: O(degree)
- List all neighbours: O(degree)
- Best for: sparse graphs (most real-world graphs)

### Edge List

A flat list of (u, v) pairs, optionally with weights.

```
[(A,B), (A,C), (B,C), (B,D)]
```

- Space: O(E)
- Best for: algorithms that iterate over all edges (Kruskal's)

### Comparison

| | Adjacency Matrix | Adjacency List | Edge List |
|-|-------------------|---------------|-----------|
| Space | O(V²) | O(V + E) | O(E) |
| Add edge | O(1) | O(1) | O(1) |
| Check edge | O(1) | O(deg) | O(E) |
| Iterate neighbours | O(V) | O(deg) | O(E) |
| Dense graphs | ✓ | | |
| Sparse graphs | | ✓ | ✓ |

---

## Traversals

### Breadth-First Search (BFS)

Explores all neighbours at the current depth before moving deeper. Uses a **queue**.

```
BFS(start):
    queue = [start]
    visited = {start}
    while queue is not empty:
        v = queue.dequeue()
        process(v)
        for each neighbour u of v:
            if u not in visited:
                visited.add(u)
                queue.enqueue(u)
```

**Properties:**
- Time: O(V + E)
- Space: O(V) for the queue and visited set
- Finds **shortest path** in unweighted graphs
- Explores level by level (distance 1, then 2, then 3, ...)

**Applications:** shortest path in unweighted graphs, web crawling, social network "degrees of separation", flood fill.

### Depth-First Search (DFS)

Explores as deep as possible before backtracking. Uses a **stack** (or recursion).

```
DFS(start):
    stack = [start]
    visited = {start}
    while stack is not empty:
        v = stack.pop()
        process(v)
        for each neighbour u of v:
            if u not in visited:
                visited.add(u)
                stack.push(u)
```

**Properties:**
- Time: O(V + E)
- Space: O(V) for the stack
- Does **not** find shortest paths
- Useful for exploring structure (components, cycles, topological order)

**Applications:** cycle detection, topological sort, connected components, maze solving, detecting strongly connected components (Tarjan's, Kosaraju's).

### BFS vs DFS

| Feature | BFS | DFS |
|---------|-----|-----|
| Data structure | Queue (FIFO) | Stack (LIFO) / recursion |
| Explores | Level by level | Branch by branch |
| Shortest path (unweighted) | ✓ | ✗ |
| Memory (worst case) | O(V) — can be wide | O(V) — can be deep |
| Cycle detection | Possible | Natural |
| Topological sort | Kahn's algorithm | Post-order DFS |

---

## Trees

A **tree** is a connected, acyclic graph. Equivalently, a graph where any two vertices are connected by exactly one path.

### Properties

For a tree with n vertices:
- Exactly n - 1 edges
- Removing any edge disconnects the graph
- Adding any edge creates exactly one cycle
- There is exactly one path between any two vertices

### Rooted Trees

A tree with a designated **root** vertex. Common specialisations:

| Type | Constraint |
|------|-----------|
| Binary tree | Each node has ≤ 2 children |
| Complete binary tree | All levels full except possibly the last |
| Full binary tree | Every node has 0 or 2 children |
| Balanced binary tree | Height ≤ O(log n) |
| k-ary tree | Each node has ≤ k children |

### Spanning Trees

A **spanning tree** of a connected graph G is a subgraph that includes all vertices and is a tree. Every connected graph has at least one spanning tree.

---

## Shortest Path Algorithms

### Dijkstra's Algorithm

Finds shortest paths from a source to all vertices in a graph with **non-negative** edge weights.

```
Dijkstra(graph, source):
    dist = {v: ∞ for v in graph}
    dist[source] = 0
    priority_queue = [(0, source)]

    while priority_queue:
        d, u = extract_min(priority_queue)
        if d > dist[u]: continue
        for (v, weight) in neighbours(u):
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                insert(priority_queue, (dist[v], v))
    return dist
```

- Time: O((V + E) log V) with binary heap
- Fails with negative edges

### Bellman-Ford Algorithm

Handles **negative** edge weights. Detects negative cycles.

- Time: O(V × E)
- Relax all edges V-1 times
- If any edge can still be relaxed on the Vth pass, a negative cycle exists

### Floyd-Warshall Algorithm

Finds shortest paths between **all pairs** of vertices.

- Time: O(V³)
- Space: O(V²)
- Dynamic programming on intermediate vertices

### Comparison

| Algorithm | Negative edges | All pairs | Time |
|-----------|---------------|-----------|------|
| Dijkstra | ✗ | Single source | O((V+E) log V) |
| Bellman-Ford | ✓ | Single source | O(VE) |
| Floyd-Warshall | ✓ (no neg. cycles) | ✓ | O(V³) |
| A* | ✗ | Single target | O((V+E) log V) |

---

## Minimum Spanning Trees

A spanning tree with minimum total edge weight. Used for network design, clustering, and approximation algorithms.

### Kruskal's Algorithm

1. Sort all edges by weight
2. Add edges in order, skipping any that would create a cycle (use union-find)
3. Stop when the tree has V-1 edges

Time: O(E log E)

### Prim's Algorithm

1. Start from any vertex
2. Repeatedly add the cheapest edge connecting the tree to a non-tree vertex
3. Use a priority queue for efficiency

Time: O((V + E) log V)

---

## Special Topics

### Euler Paths and Circuits

An **Euler path** visits every **edge** exactly once. An **Euler circuit** is an Euler path that starts and ends at the same vertex.

| Graph type | Euler path exists when | Euler circuit exists when |
|-----------|----------------------|--------------------------|
| Undirected | Exactly 0 or 2 vertices with odd degree | All vertices have even degree |
| Directed | At most 1 vertex with (out-in)=1, at most 1 with (in-out)=1 | Every vertex has equal in-degree and out-degree |

**Application:** The original graph theory problem (Königsberg bridges, 1736). Modern: circuit board routing, DNA sequencing (de Bruijn graphs).

### Hamilton Paths

A **Hamilton path** visits every **vertex** exactly once. Determining if one exists is NP-complete — no known polynomial algorithm.

**Application:** Travelling Salesman Problem (TSP), scheduling.

### Graph Colouring

Assign colours to vertices such that no two adjacent vertices share a colour. The **chromatic number** χ(G) is the minimum colours needed.

| Graph | χ(G) |
|-------|------|
| Tree | 2 |
| Bipartite | 2 |
| Odd cycle | 3 |
| Complete Kₙ | n |
| Planar graph | ≤ 4 (Four Colour Theorem) |

**Applications:** register allocation (compilers), scheduling (exam timetabling), map colouring, frequency assignment.

### Network Flow

Model a graph as a network with capacities on edges. The **max-flow min-cut theorem** states that the maximum flow from source to sink equals the minimum cut capacity separating them.

**Applications:** bandwidth allocation, bipartite matching, image segmentation, supply chain optimisation.

---

## Graphs in Engineering

| Application | Graph Model |
|-------------|-------------|
| Social networks | Undirected graph (Facebook) or directed (Twitter) |
| Web pages | Directed graph (hyperlinks), PageRank |
| Road networks | Weighted graph, Dijkstra/A* |
| Dependency management | DAG, topological sort (npm, pip, Terraform) |
| Version control (Git) | DAG of commits |
| State machines | Directed graph of states and transitions |
| Kubernetes networking | Graph of pods, services, ingress |
| Database schemas | ER diagrams as graphs, foreign key dependencies |
| Circuit design | Graph of logic gates |
| Compiler optimisation | Control flow graphs, data flow graphs |

---

## Key Takeaways

- Graphs model any relationship between objects. Choose the right representation (adjacency list for sparse, matrix for dense).
- BFS finds shortest unweighted paths and explores level-by-level. DFS explores deeply and is natural for cycle detection and topological sort.
- Trees are connected acyclic graphs with exactly V-1 edges — the simplest connected structure.
- Dijkstra's for non-negative weights, Bellman-Ford for negative weights, Floyd-Warshall for all pairs.
- Graph theory concepts (colouring, flow, Euler/Hamilton paths) have direct applications in scheduling, networking, and compiler design.
- Nearly every dependency system (package managers, build tools, CI/CD) is a DAG with topological sort.
