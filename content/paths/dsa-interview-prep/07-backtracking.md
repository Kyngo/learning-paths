---
title: "Backtracking"
weight: 7
---

# Backtracking

Backtracking systematically explores all possible solutions by making choices, checking constraints, and undoing choices that lead to dead ends. It's controlled brute force with pruning.

## Pattern Summary

| Problem Type | Time | Space | Pruning Possible? |
|-------------|------|-------|-------------------|
| Permutations | O(n!) | O(n) | Yes (used elements) |
| Combinations | O(C(n,k)) | O(k) | Yes (remaining count) |
| Subsets | O(2ⁿ) | O(n) | Minimal |
| Constraint Satisfaction | Varies | O(n²) | Yes (constraint checking) |
| String Partitioning | O(n · 2ⁿ) | O(n) | Yes (validity check) |

---

## The Backtracking Template

Every backtracking problem follows this structure:

```python
def backtrack(candidates, path, result, start=0):
    """
    candidates: available choices
    path: current partial solution
    result: collection of valid solutions
    start: index to avoid revisiting (for combinations)
    """
    # Base case: found a valid solution
    if is_solution(path):
        result.append(path[:])  # copy!
        return

    for i in range(start, len(candidates)):
        # Pruning: skip invalid choices early
        if not is_valid(candidates[i], path):
            continue

        # Make choice
        path.append(candidates[i])

        # Recurse
        backtrack(candidates, path, result, i + 1)  # or i for reuse

        # Undo choice (backtrack)
        path.pop()
```

### Decision Framework

| Question | Determines |
|----------|-----------|
| Can we reuse elements? | `start = i` (reuse) vs `start = i + 1` (no reuse) |
| Does order matter? | Permutation (no start param) vs combination (use start) |
| Are there duplicates in input? | Need sorting + skip logic |
| When is a path complete? | Base case condition |

---

## Permutations

### All Permutations (No Duplicates)

```python
def permute(nums: list[int]) -> list[list[int]]:
    """Generate all permutations of nums."""
    result = []

    def backtrack(path, remaining):
        if not remaining:
            result.append(path[:])
            return

        for i in range(len(remaining)):
            path.append(remaining[i])
            backtrack(path, remaining[:i] + remaining[i+1:])
            path.pop()

    backtrack([], nums)
    return result
```

### Permutations with Duplicates

```python
def permute_unique(nums: list[int]) -> list[list[int]]:
    """Permutations without duplicate results."""
    nums.sort()
    result = []
    used = [False] * len(nums)

    def backtrack(path):
        if len(path) == len(nums):
            result.append(path[:])
            return

        for i in range(len(nums)):
            if used[i]:
                continue
            # Skip duplicates: same value as previous AND previous not used
            if i > 0 and nums[i] == nums[i-1] and not used[i-1]:
                continue

            used[i] = True
            path.append(nums[i])
            backtrack(path)
            path.pop()
            used[i] = False

    backtrack([])
    return result
```

---

## Combinations

### Choose K from N

```python
def combine(n: int, k: int) -> list[list[int]]:
    """All combinations of k numbers from [1, n]."""
    result = []

    def backtrack(start, path):
        if len(path) == k:
            result.append(path[:])
            return

        # Pruning: need (k - len(path)) more elements
        # Can't start beyond n - (k - len(path)) + 1
        for i in range(start, n - (k - len(path)) + 2):
            path.append(i)
            backtrack(i + 1, path)
            path.pop()

    backtrack(1, [])
    return result
```

### Combination Sum (Reuse Allowed)

```python
def combination_sum(candidates: list[int], target: int) -> list[list[int]]:
    """Combinations that sum to target. Elements can be reused."""
    result = []
    candidates.sort()

    def backtrack(start, path, remaining):
        if remaining == 0:
            result.append(path[:])
            return

        for i in range(start, len(candidates)):
            if candidates[i] > remaining:
                break  # prune: sorted, so all further too large

            path.append(candidates[i])
            backtrack(i, path, remaining - candidates[i])  # i, not i+1
            path.pop()

    backtrack(0, [], target)
    return result
```

### Combination Sum II (No Reuse, Has Duplicates)

```python
def combination_sum2(candidates: list[int], target: int) -> list[list[int]]:
    """Each number used at most once. Input may have duplicates."""
    result = []
    candidates.sort()

    def backtrack(start, path, remaining):
        if remaining == 0:
            result.append(path[:])
            return

        for i in range(start, len(candidates)):
            if candidates[i] > remaining:
                break
            # Skip duplicates at same recursion level
            if i > start and candidates[i] == candidates[i-1]:
                continue

            path.append(candidates[i])
            backtrack(i + 1, path, remaining - candidates[i])
            path.pop()

    backtrack(0, [], target)
    return result
```

---

## Subsets

### All Subsets (Power Set)

```python
def subsets(nums: list[int]) -> list[list[int]]:
    """Generate all subsets (power set)."""
    result = []

    def backtrack(start, path):
        result.append(path[:])  # every path is a valid subset

        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()

    backtrack(0, [])
    return result
```

### Subsets with Duplicates

```python
def subsets_with_dup(nums: list[int]) -> list[list[int]]:
    """Subsets without duplicate subsets."""
    nums.sort()
    result = []

    def backtrack(start, path):
        result.append(path[:])

        for i in range(start, len(nums)):
            if i > start and nums[i] == nums[i-1]:
                continue  # skip duplicates at same level
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()

    backtrack(0, [])
    return result
```

---

## N-Queens

Place N queens on an N×N board so no two attack each other.

```python
def solve_n_queens(n: int) -> list[list[str]]:
    """Find all valid N-Queens placements."""
    result = []
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col

    def backtrack(row, queens):
        if row == n:
            # Build board representation
            board = []
            for _, col in sorted(queens):
                board.append('.' * col + 'Q' + '.' * (n - col - 1))
            result.append(board)
            return

        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue  # pruning: attacked position

            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)
            queens.append((row, col))

            backtrack(row + 1, queens)

            queens.pop()
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)

    backtrack(0, [])
    return result
```

---

## Sudoku Solver

```python
def solve_sudoku(board: list[list[str]]) -> None:
    """Solve sudoku in-place using backtracking."""
    rows = [set() for _ in range(9)]
    cols = [set() for _ in range(9)]
    boxes = [set() for _ in range(9)]

    # Initialize constraint sets
    for r in range(9):
        for c in range(9):
            if board[r][c] != '.':
                num = board[r][c]
                rows[r].add(num)
                cols[c].add(num)
                boxes[(r // 3) * 3 + c // 3].add(num)

    def backtrack(pos: int) -> bool:
        if pos == 81:
            return True

        r, c = divmod(pos, 9)

        if board[r][c] != '.':
            return backtrack(pos + 1)

        box_idx = (r // 3) * 3 + c // 3
        for num in '123456789':
            if num in rows[r] or num in cols[c] or num in boxes[box_idx]:
                continue

            board[r][c] = num
            rows[r].add(num)
            cols[c].add(num)
            boxes[box_idx].add(num)

            if backtrack(pos + 1):
                return True

            board[r][c] = '.'
            rows[r].remove(num)
            cols[c].remove(num)
            boxes[box_idx].remove(num)

        return False

    backtrack(0)
```

---

## Pruning Strategies

Pruning cuts branches early, dramatically reducing runtime.

| Strategy | How | Example |
|----------|-----|---------|
| Constraint check | Skip invalid choices before recursing | N-Queens: check column/diagonal |
| Sorting + early termination | Sort input, break when value too large | Combination sum: `if candidates[i] > remaining: break` |
| Duplicate skipping | Skip same value at same recursion level | `if i > start and nums[i] == nums[i-1]: continue` |
| Remaining capacity | Don't recurse if impossible to complete | Combinations: `n - i + 1 >= k - len(path)` |
| Symmetry breaking | Exploit problem symmetry | First queen in first half of first row |

---

## Complexity Analysis

| Problem | Without Pruning | With Pruning | Notes |
|---------|----------------|--------------|-------|
| Permutations (n) | O(n · n!) | O(n!) | Must generate all |
| Combinations (n, k) | O(2ⁿ) | O(C(n,k)) | Bound by binomial coefficient |
| Subsets (n) | O(n · 2ⁿ) | O(n · 2ⁿ) | Must generate all |
| N-Queens | O(n!) | O(n!) worst | Practical runtime much better |
| Sudoku | O(9⁸¹) | O(9^m) | m = empty cells |

---

## Key Takeaways

1. **All backtracking follows one template** — learn it once, apply everywhere
2. **Choose vs permute:** use `start` parameter for combinations, `used` array for permutations
3. **Duplicates in input:** sort first, then skip `nums[i] == nums[i-1]` at same level
4. **Reuse allowed:** pass `i` instead of `i + 1` to recursion
5. **Pruning is everything** — without it, backtracking is just brute force
6. **Always copy the path** when adding to results: `result.append(path[:])` 
7. **Constraint satisfaction** (N-Queens, Sudoku) — maintain sets for O(1) validity checking
