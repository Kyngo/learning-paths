---
title: "Dynamic Programming"
weight: 10
---

# Dynamic Programming

Dynamic programming (DP) is an optimisation technique for problems with **overlapping subproblems** and **optimal substructure**. It avoids redundant computation by storing results of subproblems. DP appears in ~25% of medium/hard interview problems.

---

## Recognising DP Problems

| Signal | Example |
|--------|---------|
| "Find the minimum/maximum" | Minimum cost to reach target |
| "Count the number of ways" | Number of paths, combinations |
| "Is it possible to...?" | Can you partition into equal subsets? |
| "Find the longest/shortest" | Longest increasing subsequence |
| Choices at each step affect future options | Buy/sell stock, house robber |
| Problem has recursive structure | Fibonacci, grid paths |

---

## The Two Approaches

### Top-Down (Memoisation)

Start from the original problem, recurse, cache results:

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)
```

### Bottom-Up (Tabulation)

Build solutions from smallest subproblems up:

```python
def fib(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]
    return dp[n]
```

### Space-Optimised

When you only need the last few values:

```python
def fib(n):
    if n <= 1:
        return n
    prev, curr = 0, 1
    for _ in range(2, n + 1):
        prev, curr = curr, prev + curr
    return curr
```

---

## 1D DP Patterns

### Climbing Stairs / Fibonacci Family

```python
def climb_stairs(n):
    """Ways to reach step n, taking 1 or 2 steps at a time."""
    if n <= 2:
        return n
    prev, curr = 1, 2
    for _ in range(3, n + 1):
        prev, curr = curr, prev + curr
    return curr
```

### House Robber (Skip Pattern)

```python
def rob(nums):
    """Max money robbing non-adjacent houses."""
    if not nums:
        return 0
    prev, curr = 0, 0
    for num in nums:
        prev, curr = curr, max(curr, prev + num)
    return curr
```

### Coin Change (Unbounded Knapsack)

```python
def coin_change(coins, amount):
    """Minimum coins to make amount."""
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

---

## 2D DP Patterns

### Grid Paths

```python
def unique_paths(m, n):
    """Count paths from top-left to bottom-right (only right/down)."""
    dp = [[1] * n for _ in range(m)]
    for i in range(1, m):
        for j in range(1, n):
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1]
    return dp[m - 1][n - 1]
```

### Longest Common Subsequence (LCS)

```python
def lcs(text1, text2):
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
    return dp[m][n]
```

### 0/1 Knapsack

```python
def knapsack(weights, values, capacity):
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]
    for i in range(1, n + 1):
        for w in range(capacity + 1):
            dp[i][w] = dp[i - 1][w]  # skip item
            if weights[i - 1] <= w:
                dp[i][w] = max(dp[i][w], dp[i - 1][w - weights[i - 1]] + values[i - 1])
    return dp[n][capacity]
```

---

## State Machine DP

### Best Time to Buy and Sell Stock (Multiple Transactions)

```python
def max_profit(prices):
    """Max profit with at most k transactions."""
    held, sold = float('-inf'), 0
    for price in prices:
        held, sold = max(held, sold - price), max(sold, held + price)
    return sold
```

### With Cooldown

```python
def max_profit_cooldown(prices):
    held, sold, rest = float('-inf'), 0, 0
    for price in prices:
        held, sold, rest = max(held, rest - price), held + price, max(rest, sold)
    return max(sold, rest)
```

---

## The DP Framework

For any DP problem, define:

1. **State** — what variables describe a subproblem? (`dp[i]`, `dp[i][j]`, `dp[i][j][k]`)
2. **Transition** — how does one state relate to previous states? (the recurrence relation)
3. **Base case** — what are the smallest subproblems you can solve directly?
4. **Answer** — which state holds the final answer?

### Example: Word Break

```
State:     dp[i] = can we segment s[0:i]?
Transition: dp[i] = any(dp[j] and s[j:i] in wordDict) for j in 0..i
Base case:  dp[0] = True (empty string is segmentable)
Answer:     dp[len(s)]
```

```python
def word_break(s, word_dict):
    words = set(word_dict)
    dp = [False] * (len(s) + 1)
    dp[0] = True
    for i in range(1, len(s) + 1):
        for j in range(i):
            if dp[j] and s[j:i] in words:
                dp[i] = True
                break
    return dp[len(s)]
```

---

## Interview Strategy

1. **Identify it's DP** — overlapping subproblems + optimal substructure
2. **Define the state** — what's the minimum info needed to describe a subproblem?
3. **Write the recurrence** — relate the current state to smaller states
4. **Start with top-down** (easier to get right) → convert to bottom-up if needed
5. **Optimise space** — often only the last row/few values are needed

---

## Key Takeaways

- DP = recursion + memoisation. Bottom-up is faster (no recursion overhead); top-down is easier to write.
- The hardest part is defining the state and transition — the code follows mechanically after that.
- 1D DP: Fibonacci, climbing stairs, house robber, coin change.
- 2D DP: grid paths, LCS, knapsack, edit distance.
- State machine DP: stock trading variants (hold/sold/rest transitions).
- Space optimisation: if `dp[i]` only depends on `dp[i-1]`, you only need two variables, not an array.
