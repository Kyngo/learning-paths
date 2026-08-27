---
title: "Tries & Advanced Structures"
weight: 11
---

# Tries & Advanced Structures

Specialised data structures that solve specific problem categories more efficiently than generic ones. Tries dominate prefix problems, segment trees handle range queries, and monotonic structures optimise sliding window patterns.

---

## Trie (Prefix Tree)

A tree where each node represents a character. Paths from root to nodes form prefixes; paths to marked nodes form complete words.

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True

    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end

    def starts_with(self, prefix):
        return self._find(prefix) is not None

    def _find(self, prefix):
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node
```

### Complexity

| Operation | Time | Space |
|-----------|------|-------|
| Insert | O(m) | O(m) per word |
| Search | O(m) | — |
| Prefix check | O(m) | — |

Where m = word length.

### Classic Problems

- **Implement Trie** — the base problem
- **Word Search II** — find all words from dictionary in a grid (Trie + DFS backtracking)
- **Autocomplete / Search Suggestions** — DFS from prefix node to collect words
- **Longest Common Prefix** — traverse trie until a branch point
- **Word Dictionary with Wildcards** — `.` matches any character (DFS on trie)

### When to Use a Trie

| Use Trie | Use HashMap |
|----------|------------|
| Prefix queries ("words starting with...") | Exact match only |
| Autocomplete, spell-checking | Key-value lookup |
| Many strings share prefixes | Strings are independent |
| Need ordered traversal of strings | Order doesn't matter |

---

## Monotonic Stack

A stack that maintains elements in sorted (monotonic) order — either all increasing or all decreasing. Elements are popped when a new element breaks the monotonic property.

```python
def next_greater_element(nums):
    """For each element, find the next element that is greater."""
    result = [-1] * len(nums)
    stack = []  # stores indices, values are monotonically decreasing

    for i, num in enumerate(nums):
        while stack and nums[stack[-1]] < num:
            idx = stack.pop()
            result[idx] = num
        stack.append(i)
    return result

# [2, 1, 4, 3, 5] → [4, 4, 5, 5, -1]
```

### Classic Problems

- **Next Greater Element** — the template problem
- **Daily Temperatures** — days until a warmer day
- **Largest Rectangle in Histogram** — monotonic stack of heights
- **Trapping Rain Water** — complement of histogram problem
- **Stock Span** — consecutive days with price ≤ today

---

## Monotonic Deque

A deque maintaining a monotonic property — used for sliding window min/max in O(n):

```python
from collections import deque

def max_sliding_window(nums, k):
    """Maximum value in each window of size k."""
    result = []
    dq = deque()  # stores indices, values are monotonically decreasing

    for i, num in enumerate(nums):
        # Remove elements outside the window
        while dq and dq[0] < i - k + 1:
            dq.popleft()
        # Remove elements smaller than current (they'll never be the max)
        while dq and nums[dq[-1]] < num:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(nums[dq[0]])
    return result
```

---

## Segment Tree (Range Queries)

Answers range queries (sum, min, max) and point updates in O(log n):

```python
class SegmentTree:
    def __init__(self, nums):
        self.n = len(nums)
        self.tree = [0] * (4 * self.n)
        self._build(nums, 0, 0, self.n - 1)

    def _build(self, nums, node, start, end):
        if start == end:
            self.tree[node] = nums[start]
            return
        mid = (start + end) // 2
        self._build(nums, 2 * node + 1, start, mid)
        self._build(nums, 2 * node + 2, mid + 1, end)
        self.tree[node] = self.tree[2 * node + 1] + self.tree[2 * node + 2]

    def update(self, idx, val, node=0, start=0, end=None):
        if end is None:
            end = self.n - 1
        if start == end:
            self.tree[node] = val
            return
        mid = (start + end) // 2
        if idx <= mid:
            self.update(idx, val, 2 * node + 1, start, mid)
        else:
            self.update(idx, val, 2 * node + 2, mid + 1, end)
        self.tree[node] = self.tree[2 * node + 1] + self.tree[2 * node + 2]

    def query(self, l, r, node=0, start=0, end=None):
        if end is None:
            end = self.n - 1
        if r < start or end < l:
            return 0
        if l <= start and end <= r:
            return self.tree[node]
        mid = (start + end) // 2
        return self.query(l, r, 2 * node + 1, start, mid) + \
               self.query(l, r, 2 * node + 2, mid + 1, end)
```

### Fenwick Tree (Binary Indexed Tree) — Simpler Alternative

```python
class FenwickTree:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (n + 1)

    def update(self, i, delta):
        i += 1
        while i <= self.n:
            self.tree[i] += delta
            i += i & (-i)

    def prefix_sum(self, i):
        i += 1
        total = 0
        while i > 0:
            total += self.tree[i]
            i -= i & (-i)
        return total

    def range_sum(self, l, r):
        return self.prefix_sum(r) - (self.prefix_sum(l - 1) if l > 0 else 0)
```

### When to Use Which

| Structure | Build | Query | Update | Use |
|-----------|-------|-------|--------|-----|
| Prefix sum array | O(n) | O(1) | O(n) rebuild | Static data, range sums |
| Fenwick tree | O(n log n) | O(log n) | O(log n) | Range sums with updates |
| Segment tree | O(n) | O(log n) | O(log n) | Range min/max/sum, lazy propagation |

---

## Key Takeaways

- **Trie**: O(m) prefix operations. Use for autocomplete, word search, dictionary problems.
- **Monotonic stack**: O(n) for "next greater/smaller element" patterns. Template: pop while violation, push current.
- **Monotonic deque**: O(n) sliding window min/max. Maintain decreasing/increasing order, evict stale indices.
- **Segment tree / Fenwick tree**: O(log n) range queries with updates. Segment tree is more flexible; Fenwick tree is simpler for sums.
- These structures appear in ~10% of hard interview problems. Know the templates; don't reinvent them under pressure.
