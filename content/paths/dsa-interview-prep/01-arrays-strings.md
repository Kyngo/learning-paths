---
title: "Arrays & Strings"
weight: 1
---

# Arrays & Strings

Arrays and strings are the most common data structures in interviews. Mastering a handful of patterns covers the vast majority of problems.

## Pattern Complexity Table

| Pattern | Time | Space | When to Use |
|---------|------|-------|-------------|
| Two Pointers | O(n) | O(1) | Sorted array, pair finding, partitioning |
| Sliding Window | O(n) | O(k) | Subarray/substring with constraint |
| Prefix Sums | O(n) | O(n) | Range sum queries, subarray sum equals K |
| Kadane's Algorithm | O(n) | O(1) | Maximum subarray sum |
| Sort + Scan | O(n log n) | O(1) | When order doesn't matter, grouping |

---

## Two Pointers

The two-pointer technique uses two indices that move toward each other (or in the same direction) to solve problems in a single pass.

### When to Apply

- Array is **sorted** (or can be sorted without losing information)
- Looking for **pairs** that satisfy a condition
- Need to **partition** elements in-place
- Comparing elements from **both ends**

### Template: Opposite Direction

```python
def two_sum_sorted(nums: list[int], target: int) -> list[int]:
    """Find two numbers in a sorted array that sum to target."""
    left, right = 0, len(nums) - 1

    while left < right:
        current_sum = nums[left] + nums[right]
        if current_sum == target:
            return [left, right]
        elif current_sum < target:
            left += 1
        else:
            right -= 1

    return []  # no pair found
```

### Template: Same Direction (Fast/Slow)

```python
def remove_duplicates(nums: list[int]) -> int:
    """Remove duplicates in-place from sorted array. Return new length."""
    if not nums:
        return 0

    slow = 0
    for fast in range(1, len(nums)):
        if nums[fast] != nums[slow]:
            slow += 1
            nums[slow] = nums[fast]

    return slow + 1
```

### Three Sum (Classic Interview Problem)

```python
def three_sum(nums: list[int]) -> list[list[int]]:
    """Find all unique triplets that sum to zero."""
    nums.sort()
    result = []

    for i in range(len(nums) - 2):
        # Skip duplicates for first element
        if i > 0 and nums[i] == nums[i - 1]:
            continue

        left, right = i + 1, len(nums) - 1
        while left < right:
            total = nums[i] + nums[left] + nums[right]
            if total == 0:
                result.append([nums[i], nums[left], nums[right]])
                # Skip duplicates
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif total < 0:
                left += 1
            else:
                right -= 1

    return result
```

---

## Sliding Window

Maintains a window (subarray/substring) that expands or shrinks to find an optimal answer.

### When to Apply

- Problem mentions **subarray** or **substring**
- Looking for **longest/shortest** with a constraint
- Constraint involves a **count**, **sum**, or **set of characters**

### Template: Variable-Size Window

```python
def longest_substring_k_distinct(s: str, k: int) -> int:
    """Longest substring with at most k distinct characters."""
    from collections import defaultdict

    char_count = defaultdict(int)
    left = 0
    max_length = 0

    for right in range(len(s)):
        # Expand: add right character
        char_count[s[right]] += 1

        # Shrink: while window violates constraint
        while len(char_count) > k:
            char_count[s[left]] -= 1
            if char_count[s[left]] == 0:
                del char_count[s[left]]
            left += 1

        # Update answer
        max_length = max(max_length, right - left + 1)

    return max_length
```

### Template: Fixed-Size Window

```python
def max_sum_subarray(nums: list[int], k: int) -> int:
    """Maximum sum of any subarray of size k."""
    window_sum = sum(nums[:k])
    max_sum = window_sum

    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]
        max_sum = max(max_sum, window_sum)

    return max_sum
```

### Minimum Window Substring

```python
def min_window(s: str, t: str) -> str:
    """Smallest substring of s containing all characters of t."""
    from collections import Counter

    need = Counter(t)
    missing = len(t)
    left = 0
    start, end = 0, float('inf')

    for right in range(len(s)):
        if need[s[right]] > 0:
            missing -= 1
        need[s[right]] -= 1

        while missing == 0:  # valid window
            if right - left < end - start:
                start, end = left, right
            need[s[left]] += 1
            if need[s[left]] > 0:
                missing += 1
            left += 1

    return s[start:end + 1] if end != float('inf') else ""
```

---

## Prefix Sums

Pre-compute cumulative sums so that any range sum becomes an O(1) lookup.

### When to Apply

- Multiple **range sum queries** on the same array
- **Subarray sum equals K** (combine with hashmap)
- Need running totals or **cumulative counts**

### Template: Build and Query

```python
def build_prefix_sum(nums: list[int]) -> list[int]:
    """Build prefix sum array. prefix[i] = sum(nums[0..i-1])."""
    prefix = [0] * (len(nums) + 1)
    for i in range(len(nums)):
        prefix[i + 1] = prefix[i] + nums[i]
    return prefix

def range_sum(prefix: list[int], left: int, right: int) -> int:
    """Sum of nums[left..right] inclusive."""
    return prefix[right + 1] - prefix[left]
```

### Subarray Sum Equals K (Prefix + Hashmap)

```python
def subarray_sum(nums: list[int], k: int) -> int:
    """Count subarrays whose sum equals k."""
    count = 0
    current_sum = 0
    prefix_counts = {0: 1}  # empty prefix has sum 0

    for num in nums:
        current_sum += num
        # If (current_sum - k) was seen before, those prefixes
        # form subarrays summing to k
        count += prefix_counts.get(current_sum - k, 0)
        prefix_counts[current_sum] = prefix_counts.get(current_sum, 0) + 1

    return count
```

---

## Kadane's Algorithm

Finds the maximum sum contiguous subarray in O(n) time.

### Core Idea

At each position, decide: **extend** the current subarray or **start fresh** from here.

```python
def max_subarray(nums: list[int]) -> int:
    """Maximum sum of any contiguous subarray."""
    max_sum = nums[0]
    current = nums[0]

    for i in range(1, len(nums)):
        current = max(nums[i], current + nums[i])
        max_sum = max(max_sum, current)

    return max_sum
```

### Variant: Track Start and End Indices

```python
def max_subarray_indices(nums: list[int]) -> tuple[int, int, int]:
    """Return (max_sum, start_index, end_index)."""
    max_sum = current = nums[0]
    start = end = temp_start = 0

    for i in range(1, len(nums)):
        if nums[i] > current + nums[i]:
            current = nums[i]
            temp_start = i
        else:
            current += nums[i]

        if current > max_sum:
            max_sum = current
            start = temp_start
            end = i

    return max_sum, start, end
```

---

## String Manipulation Patterns

### Reverse Words

```python
def reverse_words(s: str) -> str:
    """Reverse word order: 'hello world' -> 'world hello'."""
    return ' '.join(s.split()[::-1])
```

### Check Palindrome (Two Pointers)

```python
def is_palindrome(s: str) -> bool:
    """Check if string is palindrome (alphanumeric only)."""
    left, right = 0, len(s) - 1

    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        if s[left].lower() != s[right].lower():
            return False
        left += 1
        right -= 1

    return True
```

### String Encoding/Decoding

```python
def compress(chars: list[str]) -> int:
    """Compress ['a','a','b','b','b'] -> ['a','2','b','3']. In-place."""
    write = 0
    read = 0

    while read < len(chars):
        char = chars[read]
        count = 0
        while read < len(chars) and chars[read] == char:
            read += 1
            count += 1

        chars[write] = char
        write += 1
        if count > 1:
            for digit in str(count):
                chars[write] = digit
                write += 1

    return write
```

---

## Key Takeaways

1. **Two pointers** reduce O(n²) brute force to O(n) on sorted data
2. **Sliding window** is the go-to for subarray/substring optimisation problems
3. **Prefix sums** turn repeated range queries from O(n) to O(1)
4. **Kadane's** is a special case of dynamic programming — "extend or restart"
5. Most array/string problems combine two patterns (e.g., prefix sum + hashmap)
6. Always ask: "Can I sort first?" — sorting often unlocks simpler solutions
7. In-place modifications typically use the slow/fast pointer technique
