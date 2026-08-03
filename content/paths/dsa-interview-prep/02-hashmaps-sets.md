---
title: "Hashmaps & Sets"
weight: 2
---

# Hashmaps & Sets

Hash-based structures provide O(1) average-case lookup, making them the first tool to reach for when you need to check existence, count frequencies, or group elements.

## Pattern Summary

| Pattern | Time | Space | When to Use |
|---------|------|-------|-------------|
| Frequency Counting | O(n) | O(k) | Count occurrences, find duplicates, majority element |
| Two-Sum (Complement) | O(n) | O(n) | Find pair summing to target (unsorted) |
| Anagram Detection | O(n) | O(1)* | Compare character frequencies |
| Grouping | O(n) | O(n) | Group elements by shared property |
| Set Operations | O(n) | O(n) | Intersection, union, difference |

*O(1) space when alphabet is fixed (e.g., 26 lowercase letters).

---

## Frequency Counting

Build a frequency map, then query it. This is the foundation of many hashmap patterns.

### Template

```python
from collections import Counter

def frequency_count(nums: list[int]) -> dict[int, int]:
    """Count occurrences of each element."""
    freq = {}
    for num in nums:
        freq[num] = freq.get(num, 0) + 1
    return freq

# Or simply:
# freq = Counter(nums)
```

### Find Duplicates

```python
def contains_duplicate(nums: list[int]) -> bool:
    """Return True if any element appears more than once."""
    seen = set()
    for num in nums:
        if num in seen:
            return True
        seen.add(num)
    return False
```

### Majority Element (Boyer-Moore Voting)

```python
def majority_element(nums: list[int]) -> int:
    """Find element appearing more than n/2 times. O(1) space."""
    candidate = None
    count = 0

    for num in nums:
        if count == 0:
            candidate = num
        count += 1 if num == candidate else -1

    return candidate
```

### Top K Frequent Elements

```python
def top_k_frequent(nums: list[int], k: int) -> list[int]:
    """Return k most frequent elements. O(n) with bucket sort."""
    freq = Counter(nums)

    # Bucket sort: index = frequency, value = list of numbers
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, count in freq.items():
        buckets[count].append(num)

    result = []
    for i in range(len(buckets) - 1, -1, -1):
        for num in buckets[i]:
            result.append(num)
            if len(result) == k:
                return result

    return result
```

---

## Two-Sum Pattern (Complement Lookup)

Instead of checking every pair O(n²), store seen values and look for the complement.

### Classic Two Sum

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    """Return indices of two numbers that sum to target."""
    seen = {}  # value -> index

    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i

    return []
```

### Four Sum II (Multiple Arrays)

```python
def four_sum_count(A: list[int], B: list[int],
                   C: list[int], D: list[int]) -> int:
    """Count tuples (i,j,k,l) where A[i]+B[j]+C[k]+D[l] == 0."""
    ab_sums = Counter(a + b for a in A for b in B)
    return sum(ab_sums.get(-(c + d), 0) for c in C for d in D)
```

### Continuous Subarray Sum

```python
def check_subarray_sum(nums: list[int], k: int) -> bool:
    """Check if subarray of size >= 2 sums to multiple of k."""
    remainder_index = {0: -1}
    running_sum = 0

    for i, num in enumerate(nums):
        running_sum += num
        remainder = running_sum % k if k != 0 else running_sum

        if remainder in remainder_index:
            if i - remainder_index[remainder] >= 2:
                return True
        else:
            remainder_index[remainder] = i

    return False
```

---

## Anagram Detection

Two strings are anagrams if they have identical character frequencies. This generalises to any "permutation match" problem.

### Check Anagram

```python
def is_anagram(s: str, t: str) -> bool:
    """Check if t is an anagram of s."""
    if len(s) != len(t):
        return False
    return Counter(s) == Counter(t)
```

### Group Anagrams

```python
def group_anagrams(strs: list[str]) -> list[list[str]]:
    """Group strings that are anagrams of each other."""
    from collections import defaultdict

    groups = defaultdict(list)
    for s in strs:
        # Sorted string is the canonical key
        key = ''.join(sorted(s))
        groups[key].append(s)

    return list(groups.values())
```

### Alternative Key: Character Count Tuple

```python
def group_anagrams_count(strs: list[str]) -> list[list[str]]:
    """Group anagrams using count tuple as key (avoids sorting)."""
    from collections import defaultdict

    groups = defaultdict(list)
    for s in strs:
        count = [0] * 26
        for c in s:
            count[ord(c) - ord('a')] += 1
        groups[tuple(count)].append(s)

    return list(groups.values())
```

### Find All Anagrams in String (Sliding Window + Hashmap)

```python
def find_anagrams(s: str, p: str) -> list[int]:
    """Find all start indices of p's anagrams in s."""
    if len(p) > len(s):
        return []

    p_count = Counter(p)
    window = Counter(s[:len(p)])
    result = []

    if window == p_count:
        result.append(0)

    for i in range(len(p), len(s)):
        # Add new character
        window[s[i]] += 1
        # Remove old character
        old = s[i - len(p)]
        window[old] -= 1
        if window[old] == 0:
            del window[old]

        if window == p_count:
            result.append(i - len(p) + 1)

    return result
```

---

## Grouping Patterns

Use a hashmap where the **key** captures the shared property and the **value** collects matching elements.

### Group by Property

```python
def group_by_length(words: list[str]) -> dict[int, list[str]]:
    """Group words by their length."""
    from collections import defaultdict

    groups = defaultdict(list)
    for word in words:
        groups[len(word)].append(word)
    return dict(groups)
```

### Longest Consecutive Sequence

```python
def longest_consecutive(nums: list[int]) -> int:
    """Length of longest consecutive element sequence. O(n)."""
    num_set = set(nums)
    longest = 0

    for num in num_set:
        # Only start counting from sequence beginning
        if num - 1 not in num_set:
            length = 1
            while num + length in num_set:
                length += 1
            longest = max(longest, length)

    return longest
```

### Isomorphic Strings

```python
def is_isomorphic(s: str, t: str) -> bool:
    """Check if characters in s can be mapped to characters in t."""
    if len(s) != len(t):
        return False

    s_to_t = {}
    t_to_s = {}

    for cs, ct in zip(s, t):
        if cs in s_to_t and s_to_t[cs] != ct:
            return False
        if ct in t_to_s and t_to_s[ct] != cs:
            return False
        s_to_t[cs] = ct
        t_to_s[ct] = cs

    return True
```

---

## Set Operations

Sets excel at existence checks and mathematical set operations.

### Intersection of Two Arrays

```python
def intersection(nums1: list[int], nums2: list[int]) -> list[int]:
    """Return unique elements common to both arrays."""
    return list(set(nums1) & set(nums2))

def intersection_with_duplicates(nums1: list[int], nums2: list[int]) -> list[int]:
    """Return common elements preserving minimum frequency."""
    count1 = Counter(nums1)
    result = []
    for num in nums2:
        if count1[num] > 0:
            result.append(num)
            count1[num] -= 1
    return result
```

### Missing / Extra Numbers

```python
def find_missing(nums: list[int]) -> int:
    """Find missing number in [0, n]. Use XOR for O(1) space."""
    n = len(nums)
    xor = n  # start with n since indices go 0..n-1
    for i in range(n):
        xor ^= i ^ nums[i]
    return xor

def find_disappeared(nums: list[int]) -> list[int]:
    """Find all numbers in [1, n] not present in array."""
    # Mark indices as negative to record presence
    for num in nums:
        idx = abs(num) - 1
        nums[idx] = -abs(nums[idx])

    return [i + 1 for i in range(len(nums)) if nums[i] > 0]
```

---

## Hashmap vs Sorting: When to Choose

| Criterion | Hashmap | Sorting |
|-----------|---------|---------|
| Time | O(n) | O(n log n) |
| Space | O(n) extra | O(1) if in-place sort allowed |
| Stability | Preserves original order | Loses original order |
| Input mutability | Does not modify input | Modifies input (or needs copy) |
| Duplicate handling | Natural (count values) | Adjacent duplicates easy |
| Follow-up queries | Instant lookups | Binary search |

**Rule of thumb:** Use hashmap when O(n) time matters and space is acceptable. Use sorting when space is critical or you need ordered output.

---

## Key Takeaways

1. **Default to hashmap** when you see "find pair", "count", or "check existence"
2. **Two-sum pattern** generalises: store what you've seen, look for the complement
3. **Anagram = same frequency map** — use sorted string or count tuple as key
4. **Grouping** problems always follow the same structure: define a key function, collect into a defaultdict
5. **Sets** are underused — they simplify intersection, union, and "has this been seen?" checks
6. **Counter** from collections is your best friend — learn its API (`most_common`, arithmetic operators)
7. When space is constrained, consider whether **sorting** can replace the hashmap
