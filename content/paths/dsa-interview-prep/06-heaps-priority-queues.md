---
title: "Heaps & Priority Queues"
weight: 6
---

# Heaps & Priority Queues

A heap gives you O(1) access to the min (or max) element and O(log n) insertion/extraction. This makes it the right choice when you repeatedly need the "best" element from a dynamic collection.

## Pattern Summary

| Pattern | Time | Space | When to Use |
|---------|------|-------|-------------|
| Top-K Elements | O(n log k) | O(k) | K largest/smallest, K most frequent |
| Merge K Sorted | O(N log k) | O(k) | Combine k sorted sources |
| Two Heaps (Median) | O(n log n) | O(n) | Running median, stream statistics |
| Heap + Lazy Delete | O(n log n) | O(n) | Sliding window median, delayed removal |
| Task Scheduling | O(n log n) | O(n) | Cooldown, interval scheduling |

*N = total elements, k = number of sources or size constraint*

---

## Python's `heapq` Module

Python only provides a **min-heap**. For max-heap, negate values.

```python
import heapq

# Min-heap operations
heap = []
heapq.heappush(heap, 5)
heapq.heappush(heap, 2)
heapq.heappush(heap, 8)
smallest = heapq.heappop(heap)  # 2
peek = heap[0]                   # 5 (next smallest)

# Build heap from list: O(n)
nums = [5, 2, 8, 1, 9]
heapq.heapify(nums)  # nums is now a valid min-heap

# Max-heap trick: negate values
max_heap = []
heapq.heappush(max_heap, -5)
largest = -heapq.heappop(max_heap)  # 5

# K smallest/largest
heapq.nsmallest(3, nums)  # [1, 2, 5]
heapq.nlargest(3, nums)   # [9, 8, 5]
```

---

## Top-K Patterns

### Kth Largest Element

```python
def find_kth_largest(nums: list[int], k: int) -> int:
    """Find kth largest using min-heap of size k. O(n log k)."""
    heap = nums[:k]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num > heap[0]:
            heapq.heapreplace(heap, num)  # pop smallest, push new

    return heap[0]
```

### Top K Frequent Elements

```python
def top_k_frequent(nums: list[int], k: int) -> list[int]:
    """Return k most frequent elements using heap."""
    from collections import Counter

    freq = Counter(nums)

    # Min-heap of size k, keyed by frequency
    return heapq.nlargest(k, freq.keys(), key=freq.get)
```

### K Closest Points to Origin

```python
def k_closest(points: list[list[int]], k: int) -> list[list[int]]:
    """K points closest to origin. Max-heap of size k."""
    # Use max-heap (negate distance) to maintain k smallest
    heap = []

    for x, y in points:
        dist = -(x * x + y * y)  # negate for max-heap
        if len(heap) < k:
            heapq.heappush(heap, (dist, x, y))
        elif dist > heap[0][0]:
            heapq.heapreplace(heap, (dist, x, y))

    return [[x, y] for _, x, y in heap]
```

### Sort Characters by Frequency

```python
def frequency_sort(s: str) -> str:
    """Sort characters by frequency (most frequent first)."""
    from collections import Counter

    freq = Counter(s)
    # Max-heap by frequency
    heap = [(-count, char) for char, count in freq.items()]
    heapq.heapify(heap)

    result = []
    while heap:
        count, char = heapq.heappop(heap)
        result.append(char * (-count))

    return ''.join(result)
```

---

## Merge K Sorted Lists

### Using Min-Heap

```python
def merge_k_lists(lists: list) -> 'ListNode':
    """Merge k sorted linked lists into one sorted list."""
    heap = []

    # Push first node from each list
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))

    dummy = ListNode(0)
    curr = dummy

    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next

        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))

    return dummy.next
```

### Merge K Sorted Arrays

```python
def merge_k_sorted_arrays(arrays: list[list[int]]) -> list[int]:
    """Merge k sorted arrays. O(N log k) where N = total elements."""
    heap = []
    result = []

    # Push first element from each array: (value, array_index, element_index)
    for i, arr in enumerate(arrays):
        if arr:
            heapq.heappush(heap, (arr[0], i, 0))

    while heap:
        val, arr_idx, elem_idx = heapq.heappop(heap)
        result.append(val)

        # Push next element from same array
        if elem_idx + 1 < len(arrays[arr_idx]):
            next_val = arrays[arr_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, arr_idx, elem_idx + 1))

    return result
```

### Smallest Range Covering Elements from K Lists

```python
def smallest_range(nums: list[list[int]]) -> list[int]:
    """Smallest range that includes at least one number from each list."""
    heap = []
    current_max = float('-inf')

    # Initialize with first element from each list
    for i, arr in enumerate(nums):
        heapq.heappush(heap, (arr[0], i, 0))
        current_max = max(current_max, arr[0])

    best_range = [float('-inf'), float('inf')]

    while len(heap) == len(nums):
        current_min, list_idx, elem_idx = heapq.heappop(heap)

        # Update best range
        if current_max - current_min < best_range[1] - best_range[0]:
            best_range = [current_min, current_max]

        # Push next from same list
        if elem_idx + 1 < len(nums[list_idx]):
            next_val = nums[list_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, list_idx, elem_idx + 1))
            current_max = max(current_max, next_val)
        else:
            break  # one list exhausted

    return best_range
```

---

## Median of Stream (Two Heaps)

Maintain a max-heap for the lower half and a min-heap for the upper half. The median is always accessible from the tops.

```python
class MedianFinder:
    """Find median from a data stream. O(log n) per add, O(1) median."""

    def __init__(self):
        self.low = []   # max-heap (negated) — lower half
        self.high = []  # min-heap — upper half

    def add_num(self, num: int) -> None:
        # Push to max-heap (lower half)
        heapq.heappush(self.low, -num)

        # Balance: max of low <= min of high
        heapq.heappush(self.high, -heapq.heappop(self.low))

        # Keep sizes: len(low) >= len(high)
        if len(self.low) < len(self.high):
            heapq.heappush(self.low, -heapq.heappop(self.high))

    def find_median(self) -> float:
        if len(self.low) > len(self.high):
            return -self.low[0]
        return (-self.low[0] + self.high[0]) / 2
```

---

## Heap Implementation (Interview Favorite)

Understanding the underlying array structure is often tested directly.

```python
class MinHeap:
    """Min-heap implementation from scratch."""

    def __init__(self):
        self.heap = []

    def push(self, val: int) -> None:
        self.heap.append(val)
        self._sift_up(len(self.heap) - 1)

    def pop(self) -> int:
        if not self.heap:
            raise IndexError("pop from empty heap")
        self._swap(0, len(self.heap) - 1)
        val = self.heap.pop()
        if self.heap:
            self._sift_down(0)
        return val

    def peek(self) -> int:
        return self.heap[0]

    def _sift_up(self, idx: int) -> None:
        parent = (idx - 1) // 2
        while idx > 0 and self.heap[idx] < self.heap[parent]:
            self._swap(idx, parent)
            idx = parent
            parent = (idx - 1) // 2

    def _sift_down(self, idx: int) -> None:
        n = len(self.heap)
        while True:
            smallest = idx
            left = 2 * idx + 1
            right = 2 * idx + 2

            if left < n and self.heap[left] < self.heap[smallest]:
                smallest = left
            if right < n and self.heap[right] < self.heap[smallest]:
                smallest = right

            if smallest == idx:
                break
            self._swap(idx, smallest)
            idx = smallest

    def _swap(self, i: int, j: int) -> None:
        self.heap[i], self.heap[j] = self.heap[j], self.heap[i]
```

---

## Task Scheduler Pattern

### Task Scheduler with Cooldown

```python
def least_interval(tasks: list[str], n: int) -> int:
    """Minimum intervals to finish all tasks with cooldown n."""
    from collections import Counter

    freq = Counter(tasks)
    max_heap = [-count for count in freq.values()]
    heapq.heapify(max_heap)

    time = 0
    cooldown = []  # (available_time, remaining_count)

    while max_heap or cooldown:
        time += 1

        if max_heap:
            count = heapq.heappop(max_heap) + 1  # execute one
            if count < 0:
                cooldown.append((time + n, count))
        
        # Check if any task is ready
        if cooldown and cooldown[0][0] == time:
            _, count = cooldown.pop(0)
            heapq.heappush(max_heap, count)

    return time
```

### Reorganize String (No Adjacent Same Characters)

```python
def reorganize_string(s: str) -> str:
    """Rearrange so no two adjacent characters are same."""
    from collections import Counter

    freq = Counter(s)
    max_heap = [(-count, char) for char, count in freq.items()]
    heapq.heapify(max_heap)

    result = []
    prev_count, prev_char = 0, ''

    while max_heap:
        count, char = heapq.heappop(max_heap)
        result.append(char)

        # Push previous back if it still has remaining
        if prev_count < 0:
            heapq.heappush(max_heap, (prev_count, prev_char))

        prev_count = count + 1  # used one
        prev_char = char

    return ''.join(result) if len(result) == len(s) else ""
```

---

## Key Takeaways

1. **Top-K** → use a heap of size k (O(n log k) beats sorting's O(n log n))
2. **Merge K sorted** → min-heap with one element per source (O(N log k))
3. **Running median** → two heaps (max-heap for lower half, min-heap for upper)
4. **Python only has min-heap** — negate values for max-heap behavior
5. **heapq.heapreplace** is faster than pop + push (single sift operation)
6. For **scheduling** problems, combine heap with a cooldown queue
7. Know how to implement a heap from scratch — parent `(i-1)//2`, children `2i+1`, `2i+2`
