---
title: "Linked Lists"
weight: 3
---

# Linked Lists

Linked list problems test pointer manipulation skills. The key insight: most solutions use combinations of just a few core techniques.

## Pattern Summary

| Pattern | Time | Space | When to Use |
|---------|------|-------|-------------|
| Fast/Slow Pointers | O(n) | O(1) | Cycle detection, middle finding, nth from end |
| Reversal | O(n) | O(1) | Reverse whole list or sub-section |
| Merge | O(n + m) | O(1) | Combine sorted lists, interleave |
| Sentinel Node | O(n) | O(1) | Simplify edge cases (empty list, head deletion) |
| Runner Technique | O(n) | O(1) | Split list, reorder |

---

## Node Definition

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

---

## Fast/Slow Pointers (Floyd's Tortoise and Hare)

Two pointers move at different speeds. The fast pointer moves 2 steps for every 1 step of the slow pointer.

### Detect Cycle

```python
def has_cycle(head: ListNode) -> bool:
    """Detect if linked list has a cycle."""
    slow = fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True

    return False
```

### Find Cycle Start

```python
def detect_cycle(head: ListNode) -> ListNode:
    """Return the node where the cycle begins, or None."""
    slow = fast = head

    # Phase 1: detect cycle
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            break
    else:
        return None

    # Phase 2: find entrance
    # Distance from head to cycle start == distance from meeting point to cycle start
    slow = head
    while slow is not fast:
        slow = slow.next
        fast = fast.next

    return slow
```

### Find Middle

```python
def find_middle(head: ListNode) -> ListNode:
    """Return middle node. For even length, returns second middle."""
    slow = fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    return slow
```

### Find Nth from End

```python
def remove_nth_from_end(head: ListNode, n: int) -> ListNode:
    """Remove nth node from end. Return new head."""
    dummy = ListNode(0, head)
    fast = slow = dummy

    # Advance fast by n + 1 steps
    for _ in range(n + 1):
        fast = fast.next

    # Move both until fast hits end
    while fast:
        slow = slow.next
        fast = fast.next

    # Skip the target node
    slow.next = slow.next.next

    return dummy.next
```

---

## Reversal Patterns

### Reverse Entire List (Iterative)

```python
def reverse_list(head: ListNode) -> ListNode:
    """Reverse a linked list in-place. Return new head."""
    prev = None
    curr = head

    while curr:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node

    return prev
```

### Reverse Entire List (Recursive)

```python
def reverse_list_recursive(head: ListNode) -> ListNode:
    """Reverse recursively. Base case: empty or single node."""
    if not head or not head.next:
        return head

    new_head = reverse_list_recursive(head.next)
    head.next.next = head
    head.next = None

    return new_head
```

### Reverse Between Positions (Partial Reversal)

```python
def reverse_between(head: ListNode, left: int, right: int) -> ListNode:
    """Reverse nodes from position left to right (1-indexed)."""
    dummy = ListNode(0, head)
    prev = dummy

    # Move to node before reversal starts
    for _ in range(left - 1):
        prev = prev.next

    # Reverse the sublist
    curr = prev.next
    for _ in range(right - left):
        next_node = curr.next
        curr.next = next_node.next
        next_node.next = prev.next
        prev.next = next_node

    return dummy.next
```

### Reverse in K-Groups

```python
def reverse_k_group(head: ListNode, k: int) -> ListNode:
    """Reverse every k nodes. Leave remainder as-is."""
    # Check if k nodes available
    count = 0
    node = head
    while node and count < k:
        node = node.next
        count += 1

    if count < k:
        return head  # not enough nodes

    # Reverse k nodes
    prev = None
    curr = head
    for _ in range(k):
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node

    # head is now last node of reversed group
    # Recursively handle rest
    head.next = reverse_k_group(curr, k)

    return prev
```

---

## Merge Operations

### Merge Two Sorted Lists

```python
def merge_two_lists(l1: ListNode, l2: ListNode) -> ListNode:
    """Merge two sorted linked lists into one sorted list."""
    dummy = ListNode(0)
    tail = dummy

    while l1 and l2:
        if l1.val <= l2.val:
            tail.next = l1
            l1 = l1.next
        else:
            tail.next = l2
            l2 = l2.next
        tail = tail.next

    tail.next = l1 or l2
    return dummy.next
```

### Merge Sort on Linked List

```python
def sort_list(head: ListNode) -> ListNode:
    """Sort linked list using merge sort. O(n log n) time, O(log n) space."""
    if not head or not head.next:
        return head

    # Split in half using fast/slow
    slow, fast = head, head.next
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    mid = slow.next
    slow.next = None  # cut the list

    left = sort_list(head)
    right = sort_list(mid)

    return merge_two_lists(left, right)
```

### Add Two Numbers (Digit Lists)

```python
def add_two_numbers(l1: ListNode, l2: ListNode) -> ListNode:
    """Add numbers represented as reversed linked lists."""
    dummy = ListNode(0)
    curr = dummy
    carry = 0

    while l1 or l2 or carry:
        val = carry
        if l1:
            val += l1.val
            l1 = l1.next
        if l2:
            val += l2.val
            l2 = l2.next

        carry, digit = divmod(val, 10)
        curr.next = ListNode(digit)
        curr = curr.next

    return dummy.next
```

---

## Sentinel (Dummy) Node

A dummy node at the head eliminates edge cases for insertion/deletion at the beginning of the list.

### When to Use

- Deleting nodes (might delete head)
- Merging lists (unknown which comes first)
- Partition operations (building new sub-lists)

### Partition List

```python
def partition(head: ListNode, x: int) -> ListNode:
    """Move all nodes < x before nodes >= x, preserving order."""
    before = before_head = ListNode(0)
    after = after_head = ListNode(0)

    while head:
        if head.val < x:
            before.next = head
            before = before.next
        else:
            after.next = head
            after = after.next
        head = head.next

    after.next = None  # terminate
    before.next = after_head.next  # connect two halves

    return before_head.next
```

---

## In-Place Manipulation

### Reorder List (L0→Ln→L1→Ln-1→...)

```python
def reorder_list(head: ListNode) -> None:
    """Reorder list in-place: first, last, second, second-last..."""
    if not head or not head.next:
        return

    # Step 1: find middle
    slow, fast = head, head
    while fast.next and fast.next.next:
        slow = slow.next
        fast = fast.next.next

    # Step 2: reverse second half
    prev = None
    curr = slow.next
    slow.next = None  # cut
    while curr:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node

    # Step 3: merge alternating
    first, second = head, prev
    while second:
        tmp1, tmp2 = first.next, second.next
        first.next = second
        second.next = tmp1
        first, second = tmp1, tmp2
```

### Palindrome Check (O(1) Space)

```python
def is_palindrome(head: ListNode) -> bool:
    """Check if linked list is palindrome using O(1) space."""
    # Find middle
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    # Reverse second half
    prev = None
    while slow:
        next_node = slow.next
        slow.next = prev
        prev = slow
        slow = next_node

    # Compare halves
    left, right = head, prev
    while right:
        if left.val != right.val:
            return False
        left = left.next
        right = right.next

    return True
```

---

## Key Takeaways

1. **Draw it out** — linked list problems are visual. Sketch the pointers before coding.
2. **Dummy nodes** eliminate 90% of edge cases (empty list, single node, head changes)
3. **Fast/slow** pointers solve cycle, middle, and split problems elegantly
4. **Reversal** is the bread and butter — practice until you can write it without thinking
5. **In-place** manipulation usually combines: find middle → reverse half → merge/interleave
6. Always consider: what happens with 0 nodes? 1 node? 2 nodes? Even/odd length?
7. **Never lose a pointer** — always save `next` before reassigning `.next`
