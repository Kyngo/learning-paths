---
title: "Algorithms"
weight: 5
---

An algorithm is a finite, well-defined sequence of steps that solves a problem. This section covers the essential algorithms every programmer should know — sorting, searching, and graph traversal — with implementations and analysis.

---

## Sorting Algorithms

### Why Sorting Matters

Sorting is a prerequisite for many other algorithms (binary search, merge operations, duplicate detection). Understanding sorting algorithms teaches fundamental algorithmic techniques: divide-and-conquer, incremental construction, and heap properties.

### Bubble Sort — The Teaching Algorithm

Repeatedly swap adjacent elements if they're in the wrong order:

```python
def bubble_sort(arr: list) -> list:
    n = len(arr)
    for i in range(n):
        swapped = False
        for j in range(0, n - i - 1):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = True
        if not swapped:
            break  # already sorted
    return arr
```

**Complexity:** O(n²) average/worst, O(n) best (already sorted with optimization).
**Use case:** Never in production. Educational only.

### Insertion Sort — Good for Small/Nearly-Sorted Data

Build the sorted array one element at a time:

```python
def insertion_sort(arr: list) -> list:
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key
    return arr
```

**Complexity:** O(n²) average/worst, O(n) best.
**Use case:** Small arrays (< 20 elements), nearly sorted data. Many library sorts use insertion sort for small partitions.

### Merge Sort — Divide and Conquer

Split in half, sort each half, merge:

```python
def merge_sort(arr: list) -> list:
    if len(arr) <= 1:
        return arr
    
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    
    return merge(left, right)

def merge(left: list, right: list) -> list:
    result = []
    i = j = 0
    
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1
    
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

**Complexity:** O(n log n) always. O(n) extra space.
**Use case:** When stability matters (equal elements keep original order), linked lists, external sorting (data doesn't fit in memory).

### Quick Sort — The Practical Champion

Pick a pivot, partition around it, recurse:

```python
def quick_sort(arr: list, low: int = 0, high: int = None) -> list:
    if high is None:
        high = len(arr) - 1
    
    if low < high:
        pivot_idx = partition(arr, low, high)
        quick_sort(arr, low, pivot_idx - 1)
        quick_sort(arr, pivot_idx + 1, high)
    
    return arr

def partition(arr: list, low: int, high: int) -> int:
    pivot = arr[high]  # choose last element as pivot
    i = low - 1
    
    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    
    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
```

**Complexity:** O(n log n) average, O(n²) worst (sorted input with bad pivot). O(log n) space (recursion stack).
**Use case:** General-purpose sorting. Most language standard libraries use a variant (introsort = quicksort + heapsort fallback).

### Comparison Summary

| Algorithm | Best | Average | Worst | Space | Stable | In-place |
|-----------|------|---------|-------|-------|--------|----------|
| Bubble | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes |
| Insertion | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | No |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Yes |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | No | Yes |

---

## Searching Algorithms

### Linear Search

Check every element sequentially:

```python
def linear_search(arr: list, target) -> int:
    for i, item in enumerate(arr):
        if item == target:
            return i
    return -1
```

**Complexity:** O(n). Works on unsorted data.

### Binary Search

Requires sorted data. Halve the search space each step:

```python
def binary_search(arr: list, target) -> int:
    low, high = 0, len(arr) - 1
    
    while low <= high:
        mid = (low + high) // 2
        
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    
    return -1  # not found
```

**Complexity:** O(log n). Requires sorted input.

### Binary Search Variants

```python
def find_first_occurrence(arr: list, target) -> int:
    """Find leftmost index of target."""
    low, high = 0, len(arr) - 1
    result = -1
    
    while low <= high:
        mid = (low + high) // 2
        if arr[mid] == target:
            result = mid
            high = mid - 1  # keep searching left
        elif arr[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    
    return result

def find_insertion_point(arr: list, target) -> int:
    """Find where target should be inserted to maintain sorted order."""
    low, high = 0, len(arr)
    
    while low < high:
        mid = (low + high) // 2
        if arr[mid] < target:
            low = mid + 1
        else:
            high = mid
    
    return low
```

### Use Case: Search in Rotated Sorted Array

```python
def search_rotated(arr: list, target: int) -> int:
    """Search in a sorted array that has been rotated (e.g., [4,5,6,7,0,1,2])."""
    low, high = 0, len(arr) - 1
    
    while low <= high:
        mid = (low + high) // 2
        
        if arr[mid] == target:
            return mid
        
        # Left half is sorted
        if arr[low] <= arr[mid]:
            if arr[low] <= target < arr[mid]:
                high = mid - 1
            else:
                low = mid + 1
        # Right half is sorted
        else:
            if arr[mid] < target <= arr[high]:
                low = mid + 1
            else:
                high = mid - 1
    
    return -1
```

---

## Graph Algorithms

### Breadth-First Search (BFS)

Explore level by level. Uses a queue. Finds shortest path in unweighted graphs.

```python
from collections import deque

def bfs(graph: dict, start: str) -> dict[str, int]:
    """Return distance from start to all reachable nodes."""
    distances = {start: 0}
    queue = deque([start])
    
    while queue:
        node = queue.popleft()
        
        for neighbor in graph[node]:
            if neighbor not in distances:
                distances[neighbor] = distances[node] + 1
                queue.append(neighbor)
    
    return distances
```

### Depth-First Search (DFS)

Explore as deep as possible before backtracking. Uses a stack (or recursion).

```python
def dfs(graph: dict, start: str) -> list[str]:
    """Return nodes in DFS order."""
    visited = set()
    order = []
    
    def explore(node):
        visited.add(node)
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                explore(neighbor)
    
    explore(start)
    return order
```

### Topological Sort

Order nodes so that for every edge u→v, u comes before v. Only works on DAGs (Directed Acyclic Graphs).

```python
def topological_sort(graph: dict) -> list[str]:
    """Kahn's algorithm — BFS-based topological sort."""
    # Calculate in-degrees
    in_degree = {node: 0 for node in graph}
    for node in graph:
        for neighbor in graph[node]:
            in_degree[neighbor] = in_degree.get(neighbor, 0) + 1
    
    # Start with nodes that have no incoming edges
    queue = deque([node for node in in_degree if in_degree[node] == 0])
    result = []
    
    while queue:
        node = queue.popleft()
        result.append(node)
        
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    
    if len(result) != len(graph):
        raise ValueError("Graph has a cycle — topological sort impossible")
    
    return result
```

**Use case:** Build systems (compile dependencies in order), task scheduling, course prerequisites.

### Dijkstra's Algorithm — Shortest Path (Weighted)

```python
import heapq

def dijkstra(graph: dict, start: str) -> dict[str, float]:
    """Find shortest distances from start to all nodes in weighted graph."""
    distances = {node: float("inf") for node in graph}
    distances[start] = 0
    heap = [(0, start)]
    
    while heap:
        dist, node = heapq.heappop(heap)
        
        if dist > distances[node]:
            continue  # already found a shorter path
        
        for neighbor, weight in graph[node]:
            new_dist = dist + weight
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                heapq.heappush(heap, (new_dist, neighbor))
    
    return distances
```

**Complexity:** O((V + E) log V) with a binary heap.
**Use case:** GPS navigation, network routing, game pathfinding.

---

## Divide and Conquer

### Pattern

1. **Divide** the problem into smaller subproblems
2. **Conquer** each subproblem recursively
3. **Combine** the solutions

### Example: Maximum Subarray (Kadane's Algorithm)

```python
def max_subarray(arr: list[int]) -> int:
    """Find contiguous subarray with largest sum — O(n)."""
    max_sum = arr[0]
    current_sum = arr[0]
    
    for i in range(1, len(arr)):
        current_sum = max(arr[i], current_sum + arr[i])
        max_sum = max(max_sum, current_sum)
    
    return max_sum

# Example: [-2, 1, -3, 4, -1, 2, 1, -5, 4]
# Answer: 6 (subarray [4, -1, 2, 1])
```

### Example: Merge K Sorted Lists

```python
import heapq

def merge_k_sorted(lists: list[list[int]]) -> list[int]:
    """Merge k sorted lists into one sorted list — O(n log k)."""
    heap = []
    
    # Initialize heap with first element from each list
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))
    
    result = []
    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)
        
        # Push next element from same list
        if elem_idx + 1 < len(lists[list_idx]):
            next_val = lists[list_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, list_idx, elem_idx + 1))
    
    return result
```

---

## Hypothetical Use Cases

### Use Case 1: E-commerce Product Ranking

```python
def rank_products(products: list[dict], user_preferences: dict) -> list[dict]:
    """Rank products by relevance score — uses sorting."""
    
    def relevance_score(product):
        score = 0
        score += product["rating"] * 10
        score += product["sales_count"] * 0.01
        score -= product["price"] * 0.1
        
        # Boost for matching preferences
        for tag in product.get("tags", []):
            if tag in user_preferences.get("interests", []):
                score += 20
        
        return score
    
    # Sort by relevance (O(n log n))
    return sorted(products, key=relevance_score, reverse=True)
```

### Use Case 2: Dependency Resolution (Topological Sort)

```python
def resolve_dependencies(packages: dict[str, list[str]]) -> list[str]:
    """Determine installation order for packages with dependencies."""
    # packages = {"app": ["framework", "database"], "framework": ["utils"], ...}
    
    order = topological_sort(packages)
    return order  # Install in this order — dependencies before dependents

# Example:
# packages = {
#     "app": ["framework", "database"],
#     "framework": ["utils", "logging"],
#     "database": ["utils"],
#     "utils": [],
#     "logging": [],
# }
# Result: ["utils", "logging", "framework", "database", "app"]
```

### Use Case 3: Network Latency Optimization (Dijkstra)

```python
def find_fastest_route(network: dict, source: str, destination: str):
    """Find lowest-latency path between two data centers."""
    distances = dijkstra(network, source)
    
    if distances[destination] == float("inf"):
        return None, "No route exists"
    
    return distances[destination], reconstruct_path(network, source, destination, distances)
```

---

## Key Takeaways

1. **Know when to use what:** Binary search for sorted data, BFS for shortest path, DFS for exhaustive exploration
2. **Sorting is O(n log n) at best** (comparison-based) — don't try to beat it without special constraints
3. **Divide and conquer** reduces problems from O(n²) to O(n log n) in many cases
4. **Graph algorithms** solve real problems: routing, scheduling, dependency resolution
5. **In practice, use library implementations** — but understand the mechanics to choose correctly
