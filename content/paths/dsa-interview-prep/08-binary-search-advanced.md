---
title: "Binary Search — Advanced Patterns"
weight: 8
---

# Binary Search — Advanced Patterns

Binary search goes far beyond sorted arrays. The key insight: whenever you can define a monotonic predicate over a search space, binary search applies. This unlocks solutions to capacity problems, optimisation tasks, and rotated-array puzzles.

---

## Pattern Summary

| Pattern | Typical Problem | Search Space | Time |
|---------|----------------|--------------|------|
| Classic sorted search | Find target in array | Array indices | O(log n) |
| Bisect on answer | Minimum capacity to ship | Answer range [lo, hi] | O(n · log(answer range)) |
| Rotated array | Search in rotated sorted array | Array indices | O(log n) |
| Minimise maximum | Split array into k subarrays | [max_element, total_sum] | O(n · log S) |
| 2D matrix search | Search row-sorted + col-sorted | Flattened index or staircase | O(log(m·n)) or O(m+n) |
| First/last occurrence | Lower bound / upper bound | Array indices | O(log n) |

---

## The Binary Search Template

Most problems use one of two forms — find the **leftmost** position where a condition becomes true:

```python
def bisect_left_condition(lo, hi, condition):
    """
    Find smallest x in [lo, hi] such that condition(x) is True.
    Assumes: condition is monotonic (False...False, True...True).
    """
    while lo < hi:
        mid = (lo + hi) // 2
        if condition(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

And the **rightmost** position where a condition holds:

```python
def bisect_right_condition(lo, hi, condition):
    """
    Find largest x in [lo, hi] such that condition(x) is True.
    Assumes: condition is monotonic (True...True, False...False).
    """
    while lo < hi:
        mid = (lo + hi + 1) // 2  # round up to avoid infinite loop
        if condition(mid):
            lo = mid
        else:
            hi = mid - 1
    return lo
```

---

## Bisect on Answer

Instead of searching within an array, search within the space of possible answers.

### Example: Capacity to Ship Packages Within D Days

Given weights and a deadline, find the minimum ship capacity:

```python
def ship_within_days(weights: list[int], days: int) -> int:
    def can_ship(capacity: int) -> bool:
        day_count = 1
        current_load = 0
        for w in weights:
            if current_load + w > capacity:
                day_count += 1
                current_load = 0
            current_load += w
        return day_count <= days

    lo = max(weights)        # must carry the heaviest single item
    hi = sum(weights)        # worst case: ship everything in one day
    while lo < hi:
        mid = (lo + hi) // 2
        if can_ship(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

**Why it works:** as capacity increases, the number of days required *decreases* monotonically. We want the smallest capacity where `days_needed <= D`.

### Example: Koko Eating Bananas

Find the minimum eating speed to finish all piles within `h` hours:

```python
import math

def min_eating_speed(piles: list[int], h: int) -> int:
    def can_finish(speed: int) -> bool:
        hours = sum(math.ceil(p / speed) for p in piles)
        return hours <= h

    lo, hi = 1, max(piles)
    while lo < hi:
        mid = (lo + hi) // 2
        if can_finish(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

---

## Rotated Sorted Array

A sorted array rotated at some pivot creates two sorted halves. The key: at least one half is always sorted.

```python
def search_rotated(nums: list[int], target: int) -> int:
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            return mid

        # Left half is sorted
        if nums[lo] <= nums[mid]:
            if nums[lo] <= target < nums[mid]:
                hi = mid - 1
            else:
                lo = mid + 1
        # Right half is sorted
        else:
            if nums[mid] < target <= nums[hi]:
                lo = mid + 1
            else:
                hi = mid - 1
    return -1
```

### Finding the Minimum in a Rotated Array

```python
def find_min_rotated(nums: list[int]) -> int:
    lo, hi = 0, len(nums) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if nums[mid] > nums[hi]:
            lo = mid + 1   # minimum is in right half
        else:
            hi = mid       # mid could be the minimum
    return nums[lo]
```

---

## Minimise the Maximum (Split Problems)

Partition an array into `k` contiguous subarrays to minimise the maximum subarray sum.

```python
def split_array(nums: list[int], k: int) -> int:
    def can_split(max_sum: int) -> bool:
        count = 1
        current = 0
        for n in nums:
            if current + n > max_sum:
                count += 1
                current = 0
            current += n
        return count <= k

    lo = max(nums)
    hi = sum(nums)
    while lo < hi:
        mid = (lo + hi) // 2
        if can_split(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

This pattern also applies to:
- **Allocate books** — minimise maximum pages assigned to a student
- **Aggressive cows** — maximise minimum distance between cows
- **Painters partition** — minimise maximum time for painters

---

## 2D Matrix Search

### Fully Sorted Matrix (row-major order)

Each row is sorted and the first element of row `i+1` > last element of row `i`. Treat as a flat sorted array:

```python
def search_matrix(matrix: list[list[int]], target: int) -> bool:
    m, n = len(matrix), len(matrix[0])
    lo, hi = 0, m * n - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        val = matrix[mid // n][mid % n]
        if val == target:
            return True
        elif val < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return False
```

### Row-Sorted and Column-Sorted Matrix (Staircase Search)

Each row sorted left-to-right, each column sorted top-to-bottom. Start from top-right corner:

```python
def search_matrix_ii(matrix: list[list[int]], target: int) -> bool:
    if not matrix:
        return False
    row, col = 0, len(matrix[0]) - 1
    while row < len(matrix) and col >= 0:
        if matrix[row][col] == target:
            return True
        elif matrix[row][col] > target:
            col -= 1
        else:
            row += 1
    return False
```

Time: O(m + n) — not O(log) but optimal for this matrix structure.

---

## Using Python's bisect Module

```python
import bisect

nums = [1, 3, 5, 5, 5, 7, 9]

bisect.bisect_left(nums, 5)     # 2  (leftmost insert position)
bisect.bisect_right(nums, 5)    # 5  (rightmost insert position)

# Count occurrences of 5
count = bisect.bisect_right(nums, 5) - bisect.bisect_left(nums, 5)  # 3
```

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Wrong loop condition `lo < hi` vs `lo <= hi` | Off-by-one, infinite loop | Match to your template |
| Not rounding up for rightmost search | Infinite loop when `lo == hi - 1` | Use `(lo + hi + 1) // 2` |
| Incorrect boundary for bisect on answer | Wrong answer | `lo = smallest possible`, `hi = largest possible` |
| Forgetting edge: single-element array | Crashes or wrong result | Test with `len(nums) == 1` |

---

## Decision Flowchart

```text
Is the search space monotonic?
├── YES → Binary search applies
│   ├── Searching within an array? → Classic binary search
│   ├── Optimising a value (min/max)? → Bisect on answer
│   │   ├── Minimise maximum → lo=max(arr), hi=sum(arr)
│   │   └── Maximise minimum → lo=0, hi=max_possible
│   ├── Rotated array? → Identify sorted half, decide direction
│   └── 2D matrix?
│       ├── Fully sorted → Flatten to 1D index
│       └── Row+col sorted → Staircase O(m+n)
└── NO → Binary search does NOT apply
```

---

## Key Takeaways

1. **Monotonic predicate** is the signal — if you can phrase "find the boundary where f(x) flips from false to true," binary search works.
2. **Bisect on answer** transforms optimisation problems into decision problems: "can we achieve X?" becomes the predicate.
3. **Rotated arrays** always have one sorted half — use that to determine which side to search.
4. **Minimise-the-maximum** problems all share the same template: binary search on the answer, greedy validation inside.
5. **Boundary setup matters** — getting `lo` and `hi` wrong means wrong answers; getting the loop condition wrong means infinite loops.
6. **Python's `bisect` module** handles the common cases; use it in interviews unless the problem specifically tests your implementation.
