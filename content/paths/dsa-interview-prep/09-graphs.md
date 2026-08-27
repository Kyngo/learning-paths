---
title: "Graphs"
weight: 9
---

# Graphs

Graph problems appear in ~20% of technical interviews. They combine traversal patterns with problem-specific insight — knowing BFS/DFS is necessary but not sufficient. This section covers the patterns that let you recognise and solve graph problems systematically.

---

## When You're Looking at a Graph Problem

Not every graph problem looks like "given a graph." Recognise the disguises:

| Problem Description | Hidden Graph |
|----|------|
| "Find if there's a path from A to B" | BFS/DFS traversal |
| "Find the shortest route" | BFS (unweighted) or Dijkstra (weighted) |
| "Detect a cycle in dependencies" | Cycle detection (DFS) |
| "Order tasks given prerequisites" | Topological sort |
| "Group connected items" | Connected components (Union-Find or DFS) |
| "Can you colour/bipartite?" | BFS/DFS 2-colouring |
| "Minimum cost to connect all nodes" | Minimum spanning tree |

---

## Pattern 1: BFS — Shortest Path in Unweighted Graphs

```python
from collections import deque

def shortest_path(graph, start, target):
    queue = deque([(start, 0)])
    visited = {start}

    while queue:
        node, dist = queue.popleft()
        if node == target:
            return dist
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, dist + 1))
    return -1  # not reachable
```

**Use BFS when:** shortest path in unweighted graph, level-order traversal, minimum steps/moves.

### Classic Problems

- **Rotting Oranges** — BFS from all rotten oranges simultaneously (multi-source BFS)
- **Word Ladder** — Transform word one letter at a time, find shortest transformation
- **01 Matrix** — Distance of each cell to nearest zero (multi-source BFS)

---

## Pattern 2: DFS — Explore All Paths, Detect Cycles

```python
def has_cycle(graph, n):
    """Detect cycle in directed graph using DFS with 3-colour marking."""
    WHITE, GRAY, BLACK = 0, 1, 2
    color = [WHITE] * n

    def dfs(node):
        color[node] = GRAY
        for neighbor in graph[node]:
            if color[neighbor] == GRAY:
                return True   # back edge = cycle
            if color[neighbor] == WHITE and dfs(neighbor):
                return True
        color[node] = BLACK
        return False

    return any(color[i] == WHITE and dfs(i) for i in range(n))
```

**Use DFS when:** detecting cycles, exploring all paths, connected components, backtracking on graphs.

---

## Pattern 3: Topological Sort

Order vertices in a DAG so that every edge u→v has u before v.

### Kahn's Algorithm (BFS-Based)

```python
from collections import deque

def topological_sort(graph, n):
    in_degree = [0] * n
    for u in range(n):
        for v in graph[u]:
            in_degree[v] += 1

    queue = deque(i for i in range(n) if in_degree[i] == 0)
    order = []

    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    return order if len(order) == n else []  # empty = cycle exists
```

**Classic problems:** Course Schedule, Build Order, Alien Dictionary.

---

## Pattern 4: Union-Find (Disjoint Set Union)

Efficiently track connected components with near-O(1) union and find:

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]

    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py:
            return False  # already connected
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        self.components -= 1
        return True

    def connected(self, x, y):
        return self.find(x) == self.find(y)
```

**Classic problems:** Number of Islands (alternative to DFS), Redundant Connection, Accounts Merge, Earliest Moment When Everyone Becomes Friends.

---

## Pattern 5: Shortest Path (Weighted)

### Dijkstra's Algorithm

```python
import heapq

def dijkstra(graph, start, n):
    dist = [float('inf')] * n
    dist[start] = 0
    heap = [(0, start)]

    while heap:
        d, u = heapq.heappop(heap)
        if d > dist[u]:
            continue
        for v, weight in graph[u]:
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                heapq.heappush(heap, (dist[v], v))
    return dist
```

**Classic problems:** Network Delay Time, Cheapest Flights Within K Stops (modified Dijkstra/Bellman-Ford), Path with Minimum Effort.

---

## Pattern 6: Grid as Graph

Many interview problems use a 2D grid as an implicit graph:

```python
def bfs_grid(grid, start_r, start_c):
    rows, cols = len(grid), len(grid[0])
    directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
    queue = deque([(start_r, start_c, 0)])
    visited = {(start_r, start_c)}

    while queue:
        r, c, dist = queue.popleft()
        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and (nr, nc) not in visited and grid[nr][nc] != 1:
                visited.add((nr, nc))
                queue.append((nr, nc, dist + 1))
```

**Classic problems:** Number of Islands (DFS/BFS flood fill), Shortest Path in Binary Matrix, Surrounded Regions, Pacific Atlantic Water Flow.

---

## Interview Approach

1. **Model the graph** — adjacency list, grid, or implicit edges?
2. **Identify the pattern** — traversal, shortest path, components, ordering?
3. **Choose the algorithm** — BFS for shortest, DFS for exploration/cycles, Union-Find for components, topological sort for ordering
4. **Handle edge cases** — disconnected graph, cycles, self-loops, empty input

---

## Key Takeaways

- BFS = shortest path (unweighted), level-order. DFS = explore all, detect cycles, backtrack.
- Topological sort = dependency ordering. If the sort is shorter than the node count, there's a cycle.
- Union-Find = O(α(n)) ≈ O(1) per operation for component tracking. Better than DFS when unions are incremental.
- Grid problems are graph problems in disguise. Use 4-directional BFS/DFS with bounds checking.
- ~70% of graph interview problems reduce to one of these 6 patterns.
