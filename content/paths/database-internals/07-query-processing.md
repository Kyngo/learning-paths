---
title: "Query Processing"
weight: 7
---

## From SQL Text to Result Set

A SQL query goes through several transformation stages before producing results. Understanding this pipeline explains why some queries are fast and others are slow — regardless of how the SQL is written.

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Parser  │ → │ Analyser │ → │ Rewriter │ → │ Planner/ │ → │ Executor │
│          │   │          │   │          │   │Optimiser │   │          │
│ SQL text │   │ Semantic │   │ View     │   │ Cost-    │   │ Runs the │
│ → parse  │   │ checks,  │   │ expansion│   │ based    │   │ chosen   │
│   tree   │   │ resolve  │   │ rule     │   │ plan     │   │ plan     │
│          │   │ names    │   │ rewrites │   │ selection│   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

---

## Parsing

The parser converts SQL text into an **abstract syntax tree (AST)**, also called a parse tree.

```sql
SELECT name, salary FROM employees WHERE dept = 'Eng' ORDER BY salary DESC;
```

```
Parse Tree:
  SelectStmt
  ├── targetList: [name, salary]
  ├── fromClause: [employees]
  ├── whereClause:
  │   └── OpExpr: dept = 'Eng'
  └── sortClause:
      └── SortBy: salary DESC
```

At this stage:
- Syntax is validated (missing commas, unmatched parentheses)
- Keywords and identifiers are tokenised
- No semantic checks yet — the parser does not know if `employees` exists

---

## Analysis (Semantic Checks)

The analyser resolves all names against the **system catalogue** (metadata about tables, columns, types, functions):

- Does the table `employees` exist?
- Does it have columns `name`, `salary`, `dept`?
- Is `dept = 'Eng'` a valid comparison (types compatible)?
- Are there implicit casts needed?

The output is a **query tree** — a semantically validated version of the parse tree with resolved references.

---

## Rewriting

The rewriter applies transformation rules:

- **View expansion:** replaces view references with their underlying queries
- **Rule system:** PostgreSQL's rule system can rewrite queries (e.g., `INSTEAD OF` rules)
- **Security policies:** Row-level security policies add additional WHERE clauses

```sql
-- Original query references a view:
SELECT * FROM active_employees;

-- After rewrite (view expanded):
SELECT * FROM employees WHERE status = 'active';
```

---

## Planning and Optimisation

The planner generates multiple candidate **execution plans** and estimates the cost of each, selecting the cheapest one.

### Plan Nodes

A plan is a tree of nodes, each performing one operation:

```
Example: SELECT e.name, d.dept_name
         FROM employees e JOIN departments d ON e.dept_id = d.id
         WHERE e.salary > 80000;

Plan tree:
┌───────────────────────┐
│  Hash Join             │
│  Join Cond: dept_id=id │
├────────┬──────────────┤
│        │              │
▼        │              ▼
┌────────────────┐  ┌──────────────┐
│ Seq Scan       │  │ Seq Scan     │
│ employees      │  │ departments  │
│ Filter:        │  │              │
│ salary > 80000 │  │              │
└────────────────┘  └──────────────┘
```

### Common Plan Nodes

| Node | Description |
|------|-------------|
| **Seq Scan** | Read every page of the table sequentially |
| **Index Scan** | Traverse B+tree index, fetch matching heap tuples |
| **Index Only Scan** | Return results from index alone (visibility map check) |
| **Bitmap Index Scan** | Build a bitmap of matching pages from index |
| **Bitmap Heap Scan** | Fetch heap pages using the bitmap |
| **Nested Loop** | For each outer row, scan inner relation |
| **Hash Join** | Build hash table on inner, probe with outer |
| **Merge Join** | Merge two sorted inputs |
| **Sort** | Sort rows (in memory or on disk via external sort) |
| **Aggregate** | Compute GROUP BY / aggregation functions |
| **Limit** | Stop after N rows |

---

## Cost Estimation

The planner assigns a **cost** to each plan node based on:

### Cost Model Parameters

| Parameter | PostgreSQL default | Meaning |
|-----------|-------------------|---------|
| `seq_page_cost` | 1.0 | Cost of reading one page sequentially |
| `random_page_cost` | 4.0 | Cost of reading one page randomly (seek) |
| `cpu_tuple_cost` | 0.01 | Cost of processing one tuple |
| `cpu_index_tuple_cost` | 0.005 | Cost of processing one index entry |
| `cpu_operator_cost` | 0.0025 | Cost of evaluating one operator |

### Cost Formula (Simplified)

```
Sequential scan cost:
  startup_cost = 0
  total_cost = (num_pages × seq_page_cost) + (num_rows × cpu_tuple_cost)

Index scan cost:
  startup_cost = index_descent_cost
  total_cost = (index_pages × random_page_cost) 
             + (matching_rows × cpu_index_tuple_cost)
             + (heap_pages × random_page_cost)
             + (matching_rows × cpu_tuple_cost)
```

A sequential scan reads pages in order (cheap per page), but reads all of them. An index scan reads fewer pages, but each read is random (expensive per page). The break-even point depends on selectivity.

---

## Cardinality Estimation

The most critical (and error-prone) part of planning. The optimiser needs to estimate **how many rows** each operation produces.

### Statistics

PostgreSQL collects statistics via `ANALYZE` (run automatically by autovacuum):

```sql
-- View statistics for a column
SELECT attname, n_distinct, most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats
WHERE tablename = 'employees' AND attname = 'dept';
```

| Statistic | Purpose |
|-----------|---------|
| `n_distinct` | Number of distinct values |
| `most_common_vals` | Most frequent values |
| `most_common_freqs` | Frequency of each common value |
| `histogram_bounds` | Distribution of non-MCV values in equal-frequency buckets |
| `correlation` | Physical ordering correlation (affects index scan cost) |
| `null_frac` | Fraction of NULL values |

### Selectivity

**Selectivity** is the estimated fraction of rows that match a predicate.

```
WHERE dept = 'Eng'

If 'Eng' is in most_common_vals with frequency 0.35:
  selectivity = 0.35
  estimated rows = 0.35 × total_rows

WHERE salary > 80000

Use histogram to find fraction of values above 80000:
  If histogram shows 30% of values are > 80000:
  selectivity = 0.30
```

### Common Estimation Errors

| Scenario | Problem |
|----------|---------|
| Correlated columns | Planner assumes independence: P(A AND B) = P(A) × P(B), but real correlation is higher |
| Skewed distributions | Uniform assumption fails for Zipf-distributed data |
| Stale statistics | `ANALYZE` hasn't run after major data changes |
| Complex expressions | Functions, casts, or computed predicates have no statistics |

Cardinality estimation errors cascade through the plan tree — a 10× underestimate at one node can cause a 100× underestimate at the next join.

---

## Join Algorithms

The planner chooses among three join algorithms based on estimated costs:

### Nested Loop Join

```
for each row r in outer_table:
    for each row s in inner_table:
        if r.key == s.key:
            emit (r, s)

Cost: O(N × M) — expensive for large tables
Best when: outer is small, inner has an index
```

```
┌────────────┐
│Nested Loop │
├─────┬──────┤
│     │      │
▼     │      ▼
outer │   Index Scan
      │   inner (index on key)
```

### Hash Join

```
// Build phase
hash_table = {}
for each row s in inner_table:
    hash_table[hash(s.key)].append(s)

// Probe phase
for each row r in outer_table:
    for each s in hash_table[hash(r.key)]:
        if r.key == s.key:
            emit (r, s)

Cost: O(N + M) — but needs memory for hash table
Best when: no useful index, both tables moderately large
```

### Sort-Merge Join

```
sorted_outer = sort(outer_table, key)
sorted_inner = sort(inner_table, key)

// Merge
i = 0; j = 0
while i < |sorted_outer| and j < |sorted_inner|:
    if sorted_outer[i].key == sorted_inner[j].key:
        emit match, advance both
    elif sorted_outer[i].key < sorted_inner[j].key:
        i++
    else:
        j++

Cost: O(N log N + M log M) for sort, O(N + M) for merge
Best when: inputs already sorted (index, previous sort) or very large
```

### Join Algorithm Comparison

| Algorithm | CPU Cost | Memory | Sorted Input Needed? | Best Scenario |
|-----------|----------|--------|---------------------|---------------|
| Nested Loop | O(N × M) | Minimal | No | Small outer, indexed inner |
| Hash Join | O(N + M) | O(smaller table) | No | Medium tables, equality joins |
| Merge Join | O(N log N + M log M) | O(sort) | Benefits from it | Large tables, already sorted |

---

## EXPLAIN Internals

`EXPLAIN` shows the chosen plan. `EXPLAIN ANALYZE` runs the query and shows actual vs estimated values.

```sql
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE e.salary > 80000;
```

```
Hash Join  (cost=3.25..45.32 rows=120 width=36) (actual time=0.08..0.42 rows=135 loops=1)
  Hash Cond: (e.dept_id = d.id)
  -> Seq Scan on employees e  (cost=0.00..35.50 rows=120 width=28) (actual time=0.01..0.18 rows=135 loops=1)
       Filter: (salary > 80000)
       Rows Removed by Filter: 365
  -> Hash  (cost=2.00..2.00 rows=10 width=16) (actual time=0.03..0.03 rows=10 loops=1)
       Buckets: 1024  Batches: 1  Memory Usage: 9kB
       -> Seq Scan on departments d  (cost=0.00..2.00 rows=10 width=16) (actual time=0.01..0.01 rows=10 loops=1)
Planning Time: 0.15 ms
Execution Time: 0.52 ms
```

### Reading EXPLAIN Output

| Field | Meaning |
|-------|---------|
| `cost=startup..total` | Estimated cost in arbitrary units (not milliseconds) |
| `rows=N` | Estimated number of rows output |
| `width=N` | Average row width in bytes |
| `actual time=start..end` | Real elapsed time in milliseconds |
| `actual rows=N` | Real row count (compare with estimated) |
| `loops=N` | How many times this node executed (nested loop iterations) |
| `Rows Removed by Filter` | Rows that didn't pass the filter |
| `Buffers: shared hit=N read=N` | Buffer pool hits vs disk reads (with `BUFFERS` option) |

### When Estimates Go Wrong

```
-> Seq Scan on orders  (cost=... rows=10) (actual ... rows=50000)
                                   ^^^^                  ^^^^^
                        Estimated 10 rows, got 50,000 — estimation is off by 5000×
```

This usually means:
- Statistics are stale (`ANALYZE` needed)
- Correlated predicates
- Functional dependency the planner cannot see

---

## Key Takeaways

- Query processing flows through **parse → analyse → rewrite → plan → execute**. The planner/optimiser is the most complex and impactful stage.
- The planner evaluates multiple candidate plans using a **cost model** based on I/O and CPU estimates. It picks the lowest-cost plan.
- **Cardinality estimation** (predicting row counts) is the Achilles heel — even small errors compound across joins and can cause the planner to choose a terrible plan.
- Three join algorithms cover different scenarios: **nested loop** (small + indexed), **hash join** (medium + equality), **merge join** (large + sorted).
- `EXPLAIN ANALYZE` is the essential diagnostic tool — always compare estimated rows with actual rows to detect planning errors.
- Stale statistics are the most common cause of bad plans. Ensure `ANALYZE` runs regularly (autovacuum handles this by default in PostgreSQL).
