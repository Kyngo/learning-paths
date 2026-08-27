---
title: "B-Trees & B+Trees"
weight: 2
---

## The Index Problem

Given a heap file with millions of rows, finding a specific row without an index requires scanning every page — a **full table scan**. B-trees and B+trees solve this by maintaining a sorted, balanced tree structure that narrows the search to a handful of page reads.

The Databases & SQL path covered *when* to use indexes. This section covers *how* they work internally.

---

## B-Tree Fundamentals

A **B-tree** (Bayer & McCreight, 1972) is a self-balancing tree where:

- Each node can hold multiple keys (not just one like a binary tree)
- All leaf nodes are at the same depth (perfectly balanced)
- A node with *k* keys has *k+1* child pointers
- Nodes map to disk pages — one page per node

### B-Tree Properties (Order *m*)

| Property | Rule |
|----------|------|
| Maximum keys per node | *m − 1* |
| Maximum children per node | *m* |
| Minimum keys (non-root internal) | ⌈*m*/2⌉ − 1 |
| Minimum children (non-root internal) | ⌈*m*/2⌉ |
| Root minimum | 1 key (or empty tree) |
| All leaves | Same depth |

For a typical 8 KB page with 8-byte keys and 6-byte pointers, *m* can be several hundred — meaning a tree of height 3–4 can index billions of rows.

### Why B-Trees for Databases?

```
Binary tree (height ≈ log₂ N):
  1 billion rows → ~30 levels → 30 page reads

B-tree (height ≈ log_m N, m ≈ 500):
  1 billion rows → ~4 levels → 4 page reads
```

Each level requires one disk page read, so minimising tree height is critical for I/O performance.

---

## B-Tree vs B+Tree

The distinction matters. Most databases (PostgreSQL, MySQL InnoDB, SQLite, SQL Server, Oracle) use **B+trees**, not B-trees.

```
B-Tree:
┌─────────────┐
│  [30 | 70]  │
├────┬────┬───┤
│    │    │   │
▼    ▼    ▼   ▼
┌──┐ ┌──┐ ┌──┐
│10│ │50│ │90│  ← Data pointers at EVERY level
│↓ │ │↓ │ │↓ │
└──┘ └──┘ └──┘

B+Tree:
          ┌─────────────┐
          │  [30 | 70]  │   ← Internal: keys + child pointers only
          └──┬────┬──┬──┘
             │    │  │
     ┌───────┘    │  └───────┐
     ▼            ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│10|20|25 │→│30|40|50 │→│70|80|90 │  ← Leaves: keys + data + sibling links
└─────────┘ └─────────┘ └─────────┘
```

| Feature | B-Tree | B+Tree |
|---------|--------|--------|
| Data pointers | At every node | Only at leaves |
| Internal node fan-out | Lower (data takes space) | Higher (more keys per node) |
| Range scans | Must traverse tree repeatedly | Follow sibling pointers at leaf level |
| All keys in leaves? | No — some only in internal nodes | Yes — leaves contain all keys |
| Tree height | Taller (lower fan-out) | Shorter (higher fan-out) |

B+trees win for databases because: (1) higher fan-out means fewer levels, (2) range queries traverse a linked list of leaves instead of climbing back up the tree.

---

## B+Tree Node Structure

### Internal Node

```
┌─────────────────────────────────────────────────┐
│                Internal Node (Page)              │
├──────────────────────────────────────────────────┤
│  Page Header │ Key Count: 3                      │
├──────────────────────────────────────────────────┤
│  P0 │ K1=30 │ P1 │ K2=70 │ P2 │ K3=120 │ P3   │
│  ↓  │       │ ↓  │       │ ↓  │        │  ↓   │
│  <30│       │30–69│      │70–119│      │ ≥120  │
└──────────────────────────────────────────────────┘
  Pn = pointer (page ID of child node)
  Kn = separator key
```

### Leaf Node

```
┌──────────────────────────────────────────────────┐
│                Leaf Node (Page)                   │
├──────────────────────────────────────────────────┤
│  Page Header │ Key Count: 4 │ Prev ← │ → Next   │
├──────────────────────────────────────────────────┤
│  K1=10 │ TID(3,1) │ K2=20 │ TID(7,5) │         │
│  K3=25 │ TID(2,0) │ K4=28 │ TID(9,2) │         │
└──────────────────────────────────────────────────┘
  TID = (page, slot) pointer to heap tuple
  Prev/Next = sibling leaf pointers for range scans
```

---

## Search

Finding key *K* in a B+tree:

```
function search(node, K):
    if node is leaf:
        binary_search for K in node.keys
        return found ? node.data[K] : NOT_FOUND
    else:
        find i where keys[i-1] ≤ K < keys[i]
        return search(child[i], K)
```

**Cost:** One page read per tree level. For a tree of height *h*, search costs exactly *h* I/Os.

Example with height 4:

```
Root (in buffer pool — likely cached)
  → Level 1 internal node    (1 I/O)
    → Level 2 internal node  (1 I/O)
      → Leaf node            (1 I/O)
        → Heap page          (1 I/O)
                              ─────
                         ~3–4 I/Os total
```

---

## Insert

Inserting key *K*:

```
function insert(tree, K, value):
    leaf = find_leaf(tree.root, K)
    
    if leaf has space:
        insert K into leaf in sorted order
        return
    
    // Leaf is full → split
    split leaf into left_leaf and right_leaf
    median = right_leaf.first_key
    
    // Promote median to parent
    insert_into_parent(leaf.parent, median, right_leaf)
    // If parent is also full, split propagates upward
    // If root splits, tree grows one level taller
```

### Page Split Visualised

```
Before insert of key 25 (leaf is full, max 4 keys):

┌───────────────┐
│ 10|15|20|30   │ ← full
└───────────────┘

After split:

        ┌────┐
        │ 20 │  ← promoted to parent
        └─┬──┘
     ┌────┴─────┐
     ▼          ▼
┌─────────┐ ┌─────────┐
│ 10 | 15 │→│20|25|30 │
└─────────┘ └─────────┘
```

Splits can cascade up the tree. If the root splits, the tree grows one level taller — this is the only way a B+tree increases in height.

---

## Delete

Deletion is more complex due to maintaining the minimum occupancy invariant.

```
function delete(tree, K):
    leaf = find_leaf(tree.root, K)
    remove K from leaf
    
    if leaf.key_count ≥ min_keys:
        return  // Done
    
    // Underflow — try redistribution first
    if sibling has spare keys:
        redistribute(leaf, sibling)
        update parent separator
    else:
        merge(leaf, sibling)
        remove separator from parent
        // Parent may underflow → recurse upward
```

### Redistribution vs Merge

```
Redistribution (sibling has spare keys):
  [10|15]  [20|25|30|35]  →  [10|15|20]  [25|30|35]
  Parent separator: 20 → 25

Merge (sibling at minimum):
  [10]  [20|25]  →  [10|20|25]
  Remove separator from parent
```

In practice, databases often use "lazy" deletion — marking entries as dead and cleaning up during compaction or vacuum, avoiding the complexity of immediate merging.

---

## Fill Factor

The **fill factor** controls how full leaf pages are during bulk inserts or index creation.

| Fill Factor | Leaf usage | Trade-off |
|-------------|-----------|-----------|
| 100% | Pages packed full | Smallest index, but any insert triggers a split |
| 90% (default in PostgreSQL) | 10% free space reserved | Room for updates to existing keys |
| 70% | 30% free space | Good for heavily updated columns |
| 50% | Half-empty pages | Rarely useful, wastes space |

```sql
-- PostgreSQL: set fill factor on index
CREATE INDEX idx_email ON users(email) WITH (fillfactor = 90);

-- Rebuild to apply new fill factor
REINDEX INDEX idx_email;
```

---

## Sibling Pointers and Range Scans

Leaf nodes form a **doubly-linked list**. This enables efficient range scans without revisiting the tree.

```
Range query: WHERE age BETWEEN 25 AND 40

1. Search tree for key=25 → land on leaf L1
2. Scan right through sibling pointers:
   L1 [22|25|27] → L2 [30|33|35] → L3 [38|40|42] → stop

No tree traversal needed after the initial search.
```

This is why B+trees dominate for `ORDER BY`, `BETWEEN`, and prefix `LIKE` queries — once you find the start, scanning is sequential I/O through the leaf chain.

---

## Bulk Loading

Building a B+tree by inserting keys one at a time is inefficient (many random writes, repeated splits). Bulk loading builds the tree bottom-up.

```
Bulk load algorithm:
1. Sort all keys
2. Fill leaf pages sequentially (respecting fill factor)
3. Build internal nodes bottom-up from leaf page boundaries
4. Result: perfectly packed tree, minimal height

One-at-a-time:  O(N log N) random I/Os
Bulk load:       O(N / B) sequential I/Os  (B = keys per page)
```

PostgreSQL uses bulk loading when `CREATE INDEX` is run on an existing table (via `tuplesort`).

---

## Practical Considerations

### Prefix Truncation

Internal nodes only need enough of a key to separate children. Instead of storing the full key `"christopher"`, an internal node might store `"chr"` if that suffices to distinguish left from right children. This increases fan-out.

### Suffix Truncation

In highly redundant indexes (e.g., indexed column with a common prefix), databases can store the shared prefix once per page and only the differing suffixes per entry.

### Non-Unique Indexes

When multiple rows share the same key value, the index appends the heap TID to make entries unique internally. This maintains sort order and enables precise lookups.

### Concurrent Access

Real B+trees need latching (lightweight locks) for concurrent readers and writers. Techniques include:

- **Latch coupling (crabbing):** acquire latch on child before releasing parent
- **B-link trees:** add a right-link pointer so readers can follow splits in progress without holding parent latches
- **Optimistic latch coupling:** traverse downward with read latches, restart with write latches only if a modification is needed

---

## Key Takeaways

- B+trees store data only in leaves and keep internal nodes slim — this maximises fan-out and minimises tree height.
- A typical B+tree of height 3–4 can index billions of rows with only 3–4 page reads per lookup.
- **Page splits** are the cost of maintaining balance on insert; **merges** (or lazy cleanup) handle deletes.
- **Fill factor** trades space for update performance — lower fill factors leave room for future inserts.
- **Sibling pointers** between leaves make range scans sequential I/O, which is why B+trees excel at ordered access.
- **Bulk loading** builds the tree bottom-up in sorted order, avoiding the overhead of individual inserts.
- Concurrent B+tree access requires latching strategies; B-link trees are a common approach to reduce latch contention.
