---
title: "Stacks & Queues"
weight: 4
---

# Stacks & Queues

Stacks (LIFO) and queues (FIFO) appear in problems involving ordering, nesting, and level-by-level processing. The monotonic stack pattern alone covers a large class of interview questions.

## Pattern Summary

| Pattern | Time | Space | When to Use |
|---------|------|-------|-------------|
| Monotonic Stack | O(n) | O(n) | Next greater/smaller element, histogram |
| Valid Parentheses | O(n) | O(n) | Matching brackets, nesting validation |
| Stack as History | O(n) | O(n) | Undo operations, path simplification |
| Queue with Max/Min | O(n) | O(n) | Sliding window max/min |
| BFS with Queue | O(V+E) | O(V) | Shortest path, level-order traversal |
| Deque Patterns | O(n) | O(k) | Sliding window, double-ended access |

---

## Monotonic Stack

A stack that maintains elements in sorted order (increasing or decreasing). As you push elements, you pop anything that violates the monotonic property.

### When to Apply

- "Next greater element"
- "Previous smaller element"
- Temperature / stock span problems
- Largest rectangle in histogram

### Template: Next Greater Element

```python
def next_greater_element(nums: list[int]) -> list[int]:
    """For each element, find the next element that is greater."""
    n = len(nums)
    result = [-1] * n
    stack = []  # stores indices

    for i in range(n):
        # Pop elements smaller than current — current is their "next greater"
        while stack and nums[stack[-1]] < nums[i]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)

    return result
```

### Daily Temperatures

```python
def daily_temperatures(temperatures: list[int]) -> list[int]:
    """Days until warmer temperature. 0 if never."""
    n = len(temperatures)
    answer = [0] * n
    stack = []  # indices of unresolved days

    for i in range(n):
        while stack and temperatures[stack[-1]] < temperatures[i]:
            prev = stack.pop()
            answer[prev] = i - prev
        stack.append(i)

    return answer
```

### Largest Rectangle in Histogram

```python
def largest_rectangle_area(heights: list[int]) -> int:
    """Find largest rectangular area in histogram."""
    stack = []  # indices of increasing heights
    max_area = 0
    heights.append(0)  # sentinel to flush remaining

    for i, h in enumerate(heights):
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    heights.pop()  # restore
    return max_area
```

### Stock Span (Previous Greater)

```python
def stock_span(prices: list[int]) -> list[int]:
    """Days since price was last higher (including today)."""
    span = []
    stack = []  # (price, span_count)

    for price in prices:
        count = 1
        while stack and stack[-1][0] <= price:
            count += stack.pop()[1]
        stack.append((price, count))
        span.append(count)

    return span
```

---

## Valid Parentheses Family

### Basic: Matching Brackets

```python
def is_valid(s: str) -> bool:
    """Check if brackets are balanced and properly nested."""
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}

    for char in s:
        if char in pairs:
            if not stack or stack[-1] != pairs[char]:
                return False
            stack.pop()
        else:
            stack.append(char)

    return len(stack) == 0
```

### Minimum Removals to Make Valid

```python
def min_remove_to_make_valid(s: str) -> str:
    """Remove minimum parentheses to make string valid."""
    s = list(s)
    stack = []  # indices of unmatched '('

    for i, char in enumerate(s):
        if char == '(':
            stack.append(i)
        elif char == ')':
            if stack:
                stack.pop()
            else:
                s[i] = ''  # unmatched ')'

    # Remaining in stack are unmatched '('
    for idx in stack:
        s[idx] = ''

    return ''.join(s)
```

### Longest Valid Parentheses

```python
def longest_valid_parentheses(s: str) -> int:
    """Length of longest valid parentheses substring."""
    stack = [-1]  # base index
    max_len = 0

    for i, char in enumerate(s):
        if char == '(':
            stack.append(i)
        else:
            stack.pop()
            if not stack:
                stack.append(i)  # new base
            else:
                max_len = max(max_len, i - stack[-1])

    return max_len
```

---

## Stack as History

The stack naturally models "undo" or "back" operations.

### Simplify Unix Path

```python
def simplify_path(path: str) -> str:
    """Simplify absolute Unix path."""
    stack = []

    for part in path.split('/'):
        if part == '..':
            if stack:
                stack.pop()
        elif part and part != '.':
            stack.append(part)

    return '/' + '/'.join(stack)
```

### Evaluate Reverse Polish Notation

```python
def eval_rpn(tokens: list[str]) -> int:
    """Evaluate expression in Reverse Polish Notation."""
    stack = []
    operators = {
        '+': lambda a, b: a + b,
        '-': lambda a, b: a - b,
        '*': lambda a, b: a * b,
        '/': lambda a, b: int(a / b),  # truncate toward zero
    }

    for token in tokens:
        if token in operators:
            b, a = stack.pop(), stack.pop()
            stack.append(operators[token](a, b))
        else:
            stack.append(int(token))

    return stack[0]
```

### Basic Calculator (with +, -, parentheses)

```python
def calculate(s: str) -> int:
    """Evaluate expression with +, -, and parentheses."""
    stack = []
    result = 0
    number = 0
    sign = 1

    for char in s:
        if char.isdigit():
            number = number * 10 + int(char)
        elif char in '+-':
            result += sign * number
            number = 0
            sign = 1 if char == '+' else -1
        elif char == '(':
            stack.append(result)
            stack.append(sign)
            result = 0
            sign = 1
        elif char == ')':
            result += sign * number
            number = 0
            result *= stack.pop()  # sign before paren
            result += stack.pop()  # result before paren

    return result + sign * number
```

---

## Queue with Max/Min (Monotonic Deque)

A deque that maintains potential maximum (or minimum) candidates for a sliding window.

### Sliding Window Maximum

```python
from collections import deque

def max_sliding_window(nums: list[int], k: int) -> list[int]:
    """Maximum element in each window of size k."""
    dq = deque()  # indices of decreasing elements
    result = []

    for i in range(len(nums)):
        # Remove elements outside window
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        # Remove smaller elements (they can never be max)
        while dq and nums[dq[-1]] < nums[i]:
            dq.pop()

        dq.append(i)

        # Window is full — record maximum
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

---

## BFS with Queues

Queues are the backbone of breadth-first search — processing nodes level by level.

### Template: Level-Order BFS

```python
from collections import deque

def bfs_levels(root) -> list[list[int]]:
    """BFS returning values grouped by level."""
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        level = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result
```

### Shortest Path in Grid (BFS)

```python
def shortest_path(grid: list[list[int]]) -> int:
    """Shortest path from top-left to bottom-right in binary grid."""
    if grid[0][0] == 1 or grid[-1][-1] == 1:
        return -1

    rows, cols = len(grid), len(grid[0])
    queue = deque([(0, 0, 1)])  # row, col, distance
    visited = {(0, 0)}
    directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]

    while queue:
        r, c, dist = queue.popleft()

        if r == rows - 1 and c == cols - 1:
            return dist

        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            if (0 <= nr < rows and 0 <= nc < cols
                    and grid[nr][nc] == 0
                    and (nr, nc) not in visited):
                visited.add((nr, nc))
                queue.append((nr, nc, dist + 1))

    return -1
```

---

## Deque Patterns

A deque (double-ended queue) supports O(1) operations at both ends.

### Design Hit Counter

```python
class HitCounter:
    """Count hits in the last 300 seconds."""

    def __init__(self):
        self.hits = deque()

    def hit(self, timestamp: int) -> None:
        self.hits.append(timestamp)

    def get_hits(self, timestamp: int) -> int:
        while self.hits and self.hits[0] <= timestamp - 300:
            self.hits.popleft()
        return len(self.hits)
```

### Implement Queue with Stacks

```python
class MyQueue:
    """FIFO queue using two LIFO stacks. Amortized O(1) per operation."""

    def __init__(self):
        self.in_stack = []
        self.out_stack = []

    def push(self, x: int) -> None:
        self.in_stack.append(x)

    def pop(self) -> int:
        self._move()
        return self.out_stack.pop()

    def peek(self) -> int:
        self._move()
        return self.out_stack[-1]

    def empty(self) -> bool:
        return not self.in_stack and not self.out_stack

    def _move(self) -> None:
        if not self.out_stack:
            while self.in_stack:
                self.out_stack.append(self.in_stack.pop())
```

---

## Key Takeaways

1. **Monotonic stack** is the single most important stack pattern — it handles "next greater/smaller" in O(n)
2. **Parentheses problems** always use a stack — the question is *what* to store (chars, indices, or counts)
3. **BFS = queue** — when you need shortest path or level-order, reach for a deque
4. **Sliding window max/min** uses a monotonic deque, not a heap (O(n) vs O(n log k))
5. **Sentinel values** (like appending 0 to histogram heights) simplify stack-based algorithms
6. When you see "evaluate expression" or "parse nested structure", think stack
7. Two stacks can simulate a queue with amortized O(1) — a common design question
