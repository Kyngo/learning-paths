---
title: "Big-O Notation and Complexity Analysis"
weight: 6
---

## What Big-O Measures

Big-O notation describes how an algorithm's resource usage (time or space) **scales** as input size grows. It captures the **upper bound** of growth rate, ignoring constants and lower-order terms.

```text
O(n) means: "grows linearly with input size"
O(n²) means: "grows quadratically with input size"
O(1) means: "constant — doesn't grow with input size"
```

### Why Constants Don't Matter

```python
# Both are O(n) — the constant factor (3 vs 1) is irrelevant at scale
def triple_loop(arr):     # 3n operations
    for x in arr: pass
    for x in arr: pass
    for x in arr: pass

def single_loop(arr):     # n operations
    for x in arr: pass
```

At n = 1,000,000: 3n = 3,000,000 vs n = 1,000,000. Both are "millions."
But O(n) vs O(n²): n = 1,000,000 vs n² = 1,000,000,000,000. That's the difference that matters.

---

## Common Complexities

### Growth Rate Comparison

| n | O(1) | O(log n) | O(n) | O(n log n) | O(n²) | O(2ⁿ) |
|---|------|----------|------|------------|-------|--------|
| 10 | 1 | 3 | 10 | 33 | 100 | 1,024 |
| 100 | 1 | 7 | 100 | 664 | 10,000 | 1.27 × 10³⁰ |
| 1,000 | 1 | 10 | 1,000 | 9,966 | 1,000,000 | ∞ |
| 1,000,000 | 1 | 20 | 1,000,000 | 19,931,568 | 10¹² | ∞ |

### O(1) — Constant Time

The operation takes the same time regardless of input size:

```python
def get_first(arr):
    return arr[0]  # O(1) — always one operation

def hash_lookup(dictionary, key):
    return dictionary[key]  # O(1) average — hash computation + one lookup

def is_even(n):
    return n % 2 == 0  # O(1) — single arithmetic operation
```

### O(log n) — Logarithmic

The input is halved (or divided by a constant) each step:

```python
def binary_search(arr, target):
    low, high = 0, len(arr) - 1
    while low <= high:          # loop runs log₂(n) times
        mid = (low + high) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            low = mid + 1       # eliminate half
        else:
            high = mid - 1      # eliminate half
    return -1
```

**Intuition:** How many times can you halve n before reaching 1? That's log₂(n).

- n = 1,000 → ~10 steps
- n = 1,000,000 → ~20 steps
- n = 1,000,000,000 → ~30 steps

### O(n) — Linear

Process each element once:

```python
def find_max(arr):
    maximum = arr[0]
    for item in arr:        # n iterations
        if item > maximum:
            maximum = item
    return maximum

def sum_all(arr):
    total = 0
    for item in arr:        # n iterations
        total += item
    return total
```

### O(n log n) — Linearithmic

Divide-and-conquer algorithms that process all elements at each level:

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])    # log n levels of recursion
    right = merge_sort(arr[mid:])   # each level processes all n elements
    return merge(left, right)       # merge is O(n)
```

**Why O(n log n):** log n levels of recursion × n work per level = n log n total.

### O(n²) — Quadratic

Nested loops over the input:

```python
def has_duplicate_naive(arr):
    for i in range(len(arr)):           # n iterations
        for j in range(i + 1, len(arr)):  # up to n iterations
            if arr[i] == arr[j]:
                return True
    return False

# Better: O(n) with a hash set
def has_duplicate(arr):
    seen = set()
    for item in arr:
        if item in seen:
            return True
        seen.add(item)
    return False
```

### O(2ⁿ) — Exponential

Each element doubles the work:

```python
def fibonacci_naive(n):
    """O(2ⁿ) — each call spawns two more calls."""
    if n <= 1:
        return n
    return fibonacci_naive(n - 1) + fibonacci_naive(n - 2)

# fibonacci(40) takes seconds
# fibonacci(50) takes minutes
# fibonacci(100) would take longer than the age of the universe
```

**Fix:** Memoization reduces to O(n):

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
# fibonacci(100) is instant
```

---

## How to Analyze Code

### Rule 1: Count the Loops

```python
# Single loop → O(n)
for i in range(n):
    do_something()

# Nested loops → O(n²)
for i in range(n):
    for j in range(n):
        do_something()

# Triple nested → O(n³)
for i in range(n):
    for j in range(n):
        for k in range(n):
            do_something()
```

### Rule 2: Halving → O(log n)

```python
# Halving the input each iteration
i = n
while i > 1:
    i = i // 2
    do_something()
# Runs log₂(n) times
```

### Rule 3: Sequential = Add, Nested = Multiply

```python
# Sequential: O(n) + O(n) = O(n)
for i in range(n):
    step_one()
for i in range(n):
    step_two()

# Nested: O(n) × O(n) = O(n²)
for i in range(n):
    for j in range(n):
        combined_step()

# Mixed: O(n) × O(log n) = O(n log n)
for i in range(n):
    binary_search(sorted_arr, i)  # O(log n) per iteration
```

### Rule 4: Drop Constants and Lower Terms

```python
# 3n² + 5n + 100 → O(n²)
# The n² term dominates as n grows

# 2ⁿ + n³ → O(2ⁿ)
# Exponential dominates polynomial
```

### Rule 5: Recursive Complexity

Use the **recurrence relation**:

```text
Merge sort: T(n) = 2T(n/2) + O(n)
            → O(n log n) (Master theorem)

Binary search: T(n) = T(n/2) + O(1)
               → O(log n)

Naive Fibonacci: T(n) = T(n-1) + T(n-2)
                 → O(2ⁿ)
```

---

## Space Complexity

Same notation, applied to memory:

```python
# O(1) space — constant extra memory
def find_max(arr):
    maximum = arr[0]  # one variable regardless of input size
    for item in arr:
        maximum = max(maximum, item)
    return maximum

# O(n) space — proportional to input
def duplicate_array(arr):
    return arr[:]  # creates a copy of size n

# O(n) space — recursion stack
def factorial(n):
    if n <= 1: return 1
    return n * factorial(n - 1)  # n stack frames

# O(log n) space — balanced recursion
def binary_search_recursive(arr, target, low, high):
    # log n stack frames (halving each time)
    ...
```

### Space-Time Trade-offs

| Approach | Time | Space | Example |
|----------|------|-------|---------|
| Brute force | O(n²) | O(1) | Check all pairs for duplicates |
| Hash set | O(n) | O(n) | Store seen elements |
| Sort first | O(n log n) | O(1)* | Sort, then check adjacent |

*In-place sort like heapsort.

---

## Amortized Analysis

Some operations are expensive occasionally but cheap on average:

### Dynamic Array Append

```text
Append to array of capacity 4:
[1, 2, 3, 4] → append(5) → allocate 8, copy 4 elements → [1, 2, 3, 4, 5, _, _, _]

Cost sequence: 1, 1, 1, 1, 5, 1, 1, 1, 9, 1, 1, 1, 1, 1, 1, 1, 17, ...
                              ↑ resize              ↑ resize

Total cost for n appends: n + n (copies) = 2n → amortized O(1) per append
```

---

## Practical Implications

### What's Fast Enough?

| Complexity | n = 10⁶ operations | Feasible? |
|------------|-------------------|-----------|
| O(1) | 1 | ✓ Instant |
| O(log n) | 20 | ✓ Instant |
| O(n) | 10⁶ | ✓ Milliseconds |
| O(n log n) | 2 × 10⁷ | ✓ Under a second |
| O(n²) | 10¹² | ✗ Hours |
| O(2ⁿ) | 10³⁰⁰⁰⁰⁰ | ✗ Heat death of universe |

**Rule of thumb:** Modern computers do ~10⁸-10⁹ simple operations per second.

- n ≤ 10⁶: O(n log n) is fine
- n ≤ 10⁴: O(n²) is fine
- n ≤ 20: O(2ⁿ) is fine

### Use Case: Choosing an Algorithm for a Feature

**Problem:** Find users who share mutual friends (social network, 10M users).

- **O(n²) approach:** Compare every pair → 10¹⁴ comparisons → days
- **O(n) approach:** For each user, hash their friends, then check intersections → seconds

The difference between a feature that ships and one that doesn't.

---

## Common Mistakes

### 1. Ignoring Hidden Complexity

```python
# Looks O(n) but is O(n²)!
result = ""
for char in large_string:
    result += char  # string concatenation creates a new string each time!

# Fix: O(n)
result = "".join(chars)
```

### 2. Assuming Hash Operations Are Always O(1)

Hash map operations are O(1) **average** but O(n) **worst case** (all keys hash to same bucket). In practice, this rarely happens with good hash functions.

### 3. Confusing Best/Average/Worst Case

```python
# Quick sort:
# Best: O(n log n) — good pivot choices
# Average: O(n log n) — random data
# Worst: O(n²) — already sorted + bad pivot selection
```

---

## Key Takeaways

1. **Big-O measures scalability**, not absolute speed
2. **Focus on the dominant term** — drop constants and lower-order terms
3. **Loops multiply** — nested loops = multiplied complexities
4. **Halving = logarithmic** — binary search, balanced trees
5. **Space matters too** — sometimes you trade memory for speed
6. **Know the practical limits** — n² is fine for n=1000, not for n=1,000,000
7. **Measure, don't guess** — profiling reveals the actual bottleneck
