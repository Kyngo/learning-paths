---
title: "Trees"
weight: 5
---

# Trees

Trees are the most frequently tested data structure in interviews. Most tree problems reduce to choosing the right traversal order and deciding what information to pass up or down the recursion.

## Pattern Summary

| Pattern | Time | Space | When to Use |
|---------|------|-------|-------------|
| DFS (Recursive) | O(n) | O(h) | Most tree problems — default choice |
| BFS (Level-order) | O(n) | O(w) | Level-by-level processing, shortest path |
| BST Operations | O(h) | O(h) | Search, insert, delete, validate |
| Construction | O(n) | O(n) | Build tree from traversal arrays |
| Path Problems | O(n) | O(h) | Sum paths, max path, diameter |

*h = height (log n balanced, n worst case), w = max width*

---

## Node Definition

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

---

## DFS Traversals

### When to Use Each Order

| Order | Visit Sequence | Use For |
|-------|---------------|---------|
| Preorder | Root → Left → Right | Copy tree, serialize, prefix expression |
| Inorder | Left → Root → Right | BST sorted order, validate BST |
| Postorder | Left → Right → Root | Delete tree, evaluate expression, bottom-up |

### Recursive Templates

```python
def preorder(root: TreeNode) -> list[int]:
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

def inorder(root: TreeNode) -> list[int]:
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

def postorder(root: TreeNode) -> list[int]:
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]
```

### Iterative Inorder (Important for Interviews)

```python
def inorder_iterative(root: TreeNode) -> list[int]:
    """Iterative inorder using explicit stack."""
    result = []
    stack = []
    curr = root

    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        result.append(curr.val)
        curr = curr.right

    return result
```

### Maximum Depth

```python
def max_depth(root: TreeNode) -> int:
    """Height of tree. Classic postorder pattern."""
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))
```

### Symmetric Tree

```python
def is_symmetric(root: TreeNode) -> bool:
    """Check if tree is mirror of itself."""
    def is_mirror(left: TreeNode, right: TreeNode) -> bool:
        if not left and not right:
            return True
        if not left or not right:
            return False
        return (left.val == right.val
                and is_mirror(left.left, right.right)
                and is_mirror(left.right, right.left))

    return is_mirror(root.left, root.right) if root else True
```

---

## BFS (Level-Order Traversal)

### Template

```python
from collections import deque

def level_order(root: TreeNode) -> list[list[int]]:
    """BFS level-by-level traversal."""
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)

    return result
```

### Zigzag Level Order

```python
def zigzag_level_order(root: TreeNode) -> list[list[int]]:
    """Alternate left-to-right and right-to-left per level."""
    if not root:
        return []

    result = []
    queue = deque([root])
    left_to_right = True

    while queue:
        level = deque()
        for _ in range(len(queue)):
            node = queue.popleft()
            if left_to_right:
                level.append(node.val)
            else:
                level.appendleft(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(list(level))
        left_to_right = not left_to_right

    return result
```

### Right Side View

```python
def right_side_view(root: TreeNode) -> list[int]:
    """Nodes visible from the right side (last node per level)."""
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        for i in range(level_size):
            node = queue.popleft()
            if i == level_size - 1:
                result.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return result
```

---

## BST Operations

### Validate BST

```python
def is_valid_bst(root: TreeNode) -> bool:
    """Check if tree is a valid Binary Search Tree."""
    def validate(node, low=float('-inf'), high=float('inf')):
        if not node:
            return True
        if node.val <= low or node.val >= high:
            return False
        return (validate(node.left, low, node.val)
                and validate(node.right, node.val, high))

    return validate(root)
```

### Search in BST

```python
def search_bst(root: TreeNode, val: int) -> TreeNode:
    """Find node with given value. O(h) time."""
    if not root or root.val == val:
        return root
    if val < root.val:
        return search_bst(root.left, val)
    return search_bst(root.right, val)
```

### Insert into BST

```python
def insert_into_bst(root: TreeNode, val: int) -> TreeNode:
    """Insert value into BST. Return root."""
    if not root:
        return TreeNode(val)
    if val < root.val:
        root.left = insert_into_bst(root.left, val)
    else:
        root.right = insert_into_bst(root.right, val)
    return root
```

### Kth Smallest in BST (Inorder)

```python
def kth_smallest(root: TreeNode, k: int) -> int:
    """Find kth smallest element using inorder traversal."""
    stack = []
    curr = root

    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        k -= 1
        if k == 0:
            return curr.val
        curr = curr.right
```

---

## Tree Construction from Traversals

### From Preorder and Inorder

```python
def build_tree(preorder: list[int], inorder: list[int]) -> TreeNode:
    """Reconstruct tree from preorder and inorder traversals."""
    if not preorder:
        return None

    # First element of preorder is root
    root_val = preorder[0]
    root = TreeNode(root_val)

    # Find root in inorder to split left/right
    mid = inorder.index(root_val)

    root.left = build_tree(preorder[1:mid + 1], inorder[:mid])
    root.right = build_tree(preorder[mid + 1:], inorder[mid + 1:])

    return root
```

### Optimised with Hashmap

```python
def build_tree_optimized(preorder: list[int], inorder: list[int]) -> TreeNode:
    """O(n) construction using index map."""
    inorder_map = {val: idx for idx, val in enumerate(inorder)}
    pre_idx = [0]

    def helper(left, right):
        if left > right:
            return None

        root_val = preorder[pre_idx[0]]
        pre_idx[0] += 1
        root = TreeNode(root_val)

        mid = inorder_map[root_val]
        root.left = helper(left, mid - 1)
        root.right = helper(mid + 1, right)

        return root

    return helper(0, len(inorder) - 1)
```

---

## Lowest Common Ancestor (LCA)

### LCA in Binary Tree

```python
def lowest_common_ancestor(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    """Find LCA of nodes p and q in binary tree."""
    if not root or root == p or root == q:
        return root

    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)

    if left and right:
        return root  # p and q are in different subtrees
    return left or right
```

### LCA in BST (Optimised)

```python
def lca_bst(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    """LCA in BST — exploit sorted property. O(h)."""
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root
```

---

## Path Problems

### Path Sum (Root to Leaf)

```python
def has_path_sum(root: TreeNode, target: int) -> bool:
    """Check if root-to-leaf path with given sum exists."""
    if not root:
        return False
    if not root.left and not root.right:
        return root.val == target
    return (has_path_sum(root.left, target - root.val)
            or has_path_sum(root.right, target - root.val))
```

### Diameter of Binary Tree

```python
def diameter_of_binary_tree(root: TreeNode) -> int:
    """Longest path between any two nodes (edges count)."""
    diameter = [0]

    def depth(node):
        if not node:
            return 0
        left = depth(node.left)
        right = depth(node.right)
        diameter[0] = max(diameter[0], left + right)
        return 1 + max(left, right)

    depth(root)
    return diameter[0]
```

### Maximum Path Sum (Any to Any)

```python
def max_path_sum(root: TreeNode) -> int:
    """Maximum sum path (can start and end anywhere)."""
    max_sum = [float('-inf')]

    def gain(node):
        if not node:
            return 0
        left = max(gain(node.left), 0)   # ignore negative paths
        right = max(gain(node.right), 0)

        # Path through this node
        max_sum[0] = max(max_sum[0], node.val + left + right)

        # Return max one-sided path for parent
        return node.val + max(left, right)

    gain(root)
    return max_sum[0]
```

---

## Key Takeaways

1. **Most tree problems are DFS** — decide if you need preorder (top-down) or postorder (bottom-up)
2. **BFS** when the problem mentions "level", "depth", or "shortest"
3. **BST inorder = sorted array** — use this for kth element, validation, and conversion
4. **LCA** is a building block — many problems reduce to finding common ancestors
5. Use a **mutable container** (`[0]` or `nonlocal`) to track global state across recursive calls
6. **Return value** from recursion should be what the parent needs; update global answer separately
7. Always handle the base case: `if not root: return ...`
