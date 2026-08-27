---
title: "Index Internals"
weight: 9
---

## Beyond B+Trees

Section 2 covered B+trees — the most common index type. But databases support many specialised index structures for different access patterns. This section explores the internal mechanics of hash indexes, GiST, GIN, BRIN, bitmap indexes, and key optimisations like index-only scans and HOT updates.

The Databases & SQL path covered *when* to create indexes. This section covers *how* they work under the hood.

---

## Hash Indexes

Hash indexes map keys to bucket numbers using a hash function. Each bucket is a page (or chain of pages) containing entries with the same hash.

```
Hash function: hash(key) → bucket_number

┌────────────────────────────────────────┐
│            Hash Index                   │
├────────┬────────┬────────┬─────────────┤
│Bucket 0│Bucket 1│Bucket 2│Bucket 3 ... │
├────────┼────────┼────────┼─────────────┤
│key=12  │key=5   │key=42  │key=7        │
│ →TID   │ →TID   │ →TID   │ →TID        │
│key=28  │        │key=90  │             │
│ →TID   │        │ →TID   │             │
└────────┴────────┴────────┴─────────────┘
```

### Properties

| Feature | Hash Index | B+Tree |
|---------|-----------|--------|
| Equality lookup (`=`) | O(1) average | O(log N) |
| Range scan (`BETWEEN`, `<`, `>`) | Not supported | Excellent |
| Ordering (`ORDER BY`) | Not supported | Supported |
| Size | Can be smaller for point lookups | Larger (tree overhead) |
| WAL support (PostgreSQL) | Yes (since v10) | Yes |

### When to Use

Hash indexes are beneficial only for **pure equality lookups** on columns that are never range-scanned. In PostgreSQL, B+trees are almost always preferred because they support both equality and range operations.

---

## GiST (Generalised Search Tree)

GiST is a framework for building balanced tree indexes over arbitrary data types. It generalises B+trees to support containment, proximity, and overlap queries.

```
GiST tree (R-tree for spatial data):

             ┌─────────────────┐
             │ Bounding box:   │
             │ (0,0)–(100,100) │
             └───────┬─────────┘
              ┌──────┴──────┐
              ▼             ▼
     ┌──────────────┐  ┌──────────────┐
     │(0,0)–(50,50) │  │(50,0)–(100,100)│
     └──────┬───────┘  └──────┬───────┘
      ┌─────┴──────┐    ┌─────┴──────┐
      ▼            ▼    ▼            ▼
   Points in     Points in  Points in  Points in
   SW quadrant   NW quad.   SE quad.   NE quadrant
```

### Supported Operations

| Data type | Operations | Example |
|-----------|-----------|---------|
| Geometry/PostGIS | Contains, intersects, distance | `WHERE geom && box '(1,1),(5,5)'` |
| Range types | Overlaps, contains, adjacent | `WHERE daterange @> '2024-01-15'::date` |
| Full-text search | Match | `WHERE tsvector @@ tsquery` |
| inet/cidr | Contains, contained by | `WHERE ip_addr <<= '10.0.0.0/8'` |

```sql
-- Create a GiST index for spatial queries
CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- Query: find all points within a bounding box
SELECT * FROM locations WHERE geom && ST_MakeEnvelope(1, 1, 5, 5);
```

### Internal Mechanics

- Each internal node stores a **bounding predicate** (e.g., bounding box for spatial data)
- Search: descend into all children whose predicate overlaps the query
- Insert: find the leaf whose predicate best fits; split if full (similar to B+tree split)
- Key difference from B+tree: search may descend into **multiple children** (not just one)

---

## GIN (Generalised Inverted Index)

GIN is an inverted index — it maps individual elements (words, array values, JSON keys) to the set of rows containing them.

```
Inverted index for full-text search:

Term        → Posting List (sorted TIDs)
"database"  → [(1,2), (3,5), (7,1), (12,4)]
"engine"    → [(1,2), (5,3), (9,0)]
"storage"   → [(2,1), (3,5), (8,2), (12,4), (15,0)]
```

### How GIN Search Works

```
Query: tsvector @@ to_tsquery('database & engine')

1. Look up "database" → posting list A
2. Look up "engine"   → posting list B
3. Intersect A ∩ B    → rows containing both terms
4. Return matching TIDs
```

### GIN Use Cases

| Data type | Index contents | Query |
|-----------|---------------|-------|
| `tsvector` | Individual lexemes | Full-text search |
| `jsonb` | Keys and values | `WHERE data @> '{"type": "order"}'` |
| `int[]`, `text[]` | Array elements | `WHERE tags @> ARRAY['urgent']` |
| `hstore` | Key-value pairs | `WHERE attrs ? 'color'` |

```sql
-- Create GIN index for JSONB
CREATE INDEX idx_orders_data ON orders USING gin(data);

-- Query: find orders with specific JSON structure
SELECT * FROM orders WHERE data @> '{"status": "shipped"}';
```

### Pending List

GIN indexes use a **pending list** (fastupdate) to batch insertions. New entries go into an unsorted pending list and are periodically merged into the main tree — this amortises the cost of maintaining the sorted posting lists.

---

## BRIN (Block Range Index)

BRIN stores summary statistics (min/max) for **ranges of consecutive pages**. Extremely compact but only useful when data is physically correlated with the indexed column.

```
BRIN with pages_per_range = 128:

Block Range     Min       Max
[0..127]        2024-01-01  2024-01-15
[128..255]      2024-01-15  2024-01-31
[256..383]      2024-02-01  2024-02-14
[384..511]      2024-02-15  2024-02-28
```

### How BRIN Search Works

```
Query: WHERE created_at > '2024-02-10'

1. Scan BRIN: which block ranges could contain rows > 2024-02-10?
   [0..127]:   max=Jan 15 → skip
   [128..255]: max=Jan 31 → skip
   [256..383]: max=Feb 14 → MAYBE (min=Feb 1, max=Feb 14)
   [384..511]: max=Feb 28 → MAYBE

2. Only scan pages in ranges [256..511]
3. Skip 50% of the table without touching it
```

### BRIN Properties

| Property | Value |
|----------|-------|
| Index size | Extremely small (few KB for millions of rows) |
| Build time | Very fast (single scan) |
| Best for | Append-only tables (timestamps, serial IDs) where physical order matches value order |
| Worst for | Randomly distributed values (every block range covers full value space) |
| Maintenance | Needs `VACUUM` or manual `brin_summarize_new_values()` for new pages |

```sql
CREATE INDEX idx_logs_created ON logs USING brin(created_at)
  WITH (pages_per_range = 128);
```

---

## Bitmap Indexes

A **bitmap index** represents each distinct value as a bit array with one bit per row.

```
Column "color" with values: Red, Blue, Green

Row#:     0  1  2  3  4  5  6  7
Red:      1  0  0  1  0  0  1  0
Blue:     0  1  0  0  1  0  0  1
Green:    0  0  1  0  0  1  0  0
```

### Bitmap Operations

Combining predicates is extremely fast — just bitwise AND/OR:

```
WHERE color = 'Red' AND size = 'Large'

Red bitmap:    1  0  0  1  0  0  1  0
Large bitmap:  0  0  0  1  1  0  1  0
AND result:    0  0  0  1  0  0  1  0
                         ↑        ↑
                     Rows 3 and 6 match
```

### PostgreSQL's Bitmap Index Scan

PostgreSQL does not store persistent bitmap indexes, but it constructs **bitmap index scans** at query time by combining results from multiple B+tree indexes:

```sql
EXPLAIN SELECT * FROM products WHERE color = 'Red' AND size = 'Large';

Bitmap Heap Scan on products
  Recheck Cond: ((color = 'Red') AND (size = 'Large'))
  -> BitmapAnd
     -> Bitmap Index Scan on idx_color (color = 'Red')
     -> Bitmap Index Scan on idx_size  (size = 'Large')
```

---

## Partial Indexes

A partial index includes only rows matching a predicate — reducing index size and maintenance cost.

```sql
CREATE INDEX idx_active_orders ON orders(customer_id)
  WHERE status = 'active';
```

```
Full index:        1,000,000 entries (all orders)
Partial index:         8,000 entries (only active orders)

The partial index is 125× smaller and 125× faster to maintain.
```

### Internals

The index B+tree is built only from qualifying tuples. During INSERT/UPDATE, the database checks whether the tuple satisfies the partial index predicate before adding an index entry.

---

## Covering Indexes and Index-Only Scans

A **covering index** includes all columns needed by a query, enabling an **index-only scan** — the database reads the answer entirely from the index without touching the heap.

```sql
CREATE INDEX idx_emp_dept_salary ON employees(dept, salary) INCLUDE (name);

-- This query can be answered from the index alone:
SELECT name, salary FROM employees WHERE dept = 'Eng';
```

### How Index-Only Scans Work

```
Normal index scan:
  1. Search index → get TID (page, slot)
  2. Fetch heap page → read full tuple
  3. Return columns from tuple

Index-only scan:
  1. Search index → all needed columns are in the index
  2. Check visibility map: is the heap page all-visible?
     Yes → return data from index (skip heap entirely)
     No  → fetch heap page (need to check tuple visibility)
```

The **visibility map** is critical for index-only scans. If the page is not all-visible, the database must visit the heap to verify tuple visibility — negating the benefit.

---

## HOT Updates (Heap-Only Tuples)

When a row is updated but **no indexed column changes**, PostgreSQL can place the new tuple version on the **same heap page** without updating any index.

```
Before HOT update:
  Index entry → (page 5, slot 2)
  Page 5, slot 2: xmin=100, xmax=0, data=(1, 'Alice', 'Eng')

After HOT update (name changed, but name is not indexed):
  Index entry → (page 5, slot 2)  ← UNCHANGED
  Page 5, slot 2: xmin=100, xmax=200, ctid→(5,4)  ← Old, points to new
  Page 5, slot 4: xmin=200, xmax=0, data=(1, 'Bob', 'Eng')  ← New version
```

### HOT Chain

The index still points to slot 2. When accessed, PostgreSQL follows the `ctid` chain within the page to find the latest visible version.

### Requirements for HOT

1. New tuple fits on the **same page** as the old tuple
2. **No indexed column** was modified

### Benefits

- No index maintenance for the update (saves I/O)
- Dead HOT tuples can be reclaimed by **micro-vacuum** (pruning within a single page) without a full vacuum
- Significant performance improvement for update-heavy workloads on tables with many indexes

---

## Index Bloat

Over time, indexes accumulate dead entries from deleted and updated rows, increasing size beyond what live data requires.

### Causes

- Frequent updates (each update creates a new index entry unless HOT applies)
- Deleted rows leave dead index entries until vacuum
- B+tree pages may be half-empty after splits and deletions

### Detection

```sql
-- Estimated bloat (PostgreSQL)
SELECT
  schemaname, tablename, indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan AS scans
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- More precise: use pgstattuple extension
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstatindex('idx_users_email');
```

### Remediation

| Action | Effect | Downtime |
|--------|--------|----------|
| `REINDEX INDEX` | Rebuilds from scratch | Locks table (blocks writes) |
| `REINDEX CONCURRENTLY` | Builds new index alongside old | No downtime (PostgreSQL 12+) |
| `VACUUM` | Removes dead index entries | No downtime |
| `pg_repack` | Rebuilds without exclusive lock | Minimal (extension required) |

---

## Key Takeaways

- **Hash indexes** offer O(1) equality lookups but cannot support range queries or ordering — B+trees are almost always a better default.
- **GiST** generalises B+trees for containment and proximity queries (spatial, ranges, full-text). Search may descend into multiple children.
- **GIN** is an inverted index ideal for multi-valued data (arrays, JSONB, full-text). The pending list batches insertions for efficiency.
- **BRIN** is extremely compact (KB-scale for millions of rows) but only works when physical row order correlates with the indexed column.
- **Bitmap index scans** combine multiple B+tree indexes at query time using bitwise AND/OR — PostgreSQL builds them dynamically rather than storing persistent bitmaps.
- **Covering indexes** with `INCLUDE` columns enable **index-only scans** that skip the heap entirely, limited by the visibility map.
- **HOT updates** avoid index maintenance when no indexed column changes — a major performance optimisation for update-heavy workloads.
- **Index bloat** is inevitable with frequent updates; monitor with `pgstattuple` and remediate with `REINDEX CONCURRENTLY` or `pg_repack`.
