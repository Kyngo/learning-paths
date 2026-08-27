---
title: "Storage Fundamentals"
weight: 1
---

## Why Storage Matters

Every database operation — reads, writes, joins, index lookups — ultimately translates into I/O against a storage device. Understanding how data is physically organised determines whether a query takes microseconds or minutes.

This section covers the lowest layer of a database: how bytes are arranged on disk, how the database divides storage into fixed-size pages, and how individual rows are packed into those pages.

---

## Disk vs SSD

### Mechanical Hard Drives (HDD)

HDDs store data on spinning magnetic platters read by a moving head. The cost of an I/O operation is dominated by **seek time** (moving the head to the right track) and **rotational latency** (waiting for the platter to spin to the right sector).

```
┌─────────────────────────────────────────┐
│           HDD I/O Breakdown             │
├─────────────────────────────────────────┤
│  Seek time         ~4–10 ms            │
│  Rotational latency ~2–6 ms            │
│  Transfer rate      ~100–200 MB/s      │
│  Random 4KB read    ~5–15 ms           │
│  Sequential read    ~100–200 MB/s      │
└─────────────────────────────────────────┘
```

Because seeks are expensive, databases on HDD are designed to minimise random I/O. Sequential access is 100–1000× faster than random access.

### Solid State Drives (SSD)

SSDs use NAND flash memory with no moving parts. Random reads are orders of magnitude faster, but writes have unique constraints.

```
┌─────────────────────────────────────────┐
│           SSD I/O Breakdown             │
├─────────────────────────────────────────┤
│  Random 4KB read    ~0.05–0.1 ms       │
│  Random 4KB write   ~0.05–0.5 ms       │
│  Sequential read    ~500–7000 MB/s     │
│  Sequential write   ~500–5000 MB/s     │
│  Write amplification factor: 2–10×     │
└─────────────────────────────────────────┘
```

### SSD Write Constraints

SSDs cannot overwrite data in place. They must **erase** a block (128–512 KB) before writing. This leads to:

| Concept | Description |
|---------|-------------|
| **Write amplification** | More physical writes than logical writes due to block erase/rewrite cycles |
| **Wear levelling** | Controller distributes writes evenly across cells to extend lifespan |
| **TRIM** | OS tells SSD which blocks are no longer in use, enabling better garbage collection |
| **Write endurance** | Each cell can be written a finite number of times (TLC: ~1000–3000 cycles) |

### Implications for Database Design

| Design choice | HDD-optimised | SSD-optimised |
|---------------|---------------|---------------|
| Page size | Larger (16–64 KB) to amortise seeks | Smaller pages acceptable (4–8 KB) |
| Random I/O | Avoid at all costs | Acceptable, but sequential still faster |
| Write pattern | Append-only / sequential | Random writes feasible but amplification matters |
| Index structure | B+tree (sequential leaf scans) | Both B+tree and LSM viable |

---

## Page-Based Storage

Databases do not read or write individual rows. They operate in fixed-size **pages** (also called blocks), typically 4 KB, 8 KB, or 16 KB. This aligns with how operating systems and storage devices handle I/O.

```
┌──────────────────────────────────────────────┐
│              Database File                    │
├──────────┬──────────┬──────────┬─────────────┤
│  Page 0  │  Page 1  │  Page 2  │  Page 3 ... │
│  (8 KB)  │  (8 KB)  │  (8 KB)  │  (8 KB)     │
└──────────┴──────────┴──────────┴─────────────┘
```

### Why Fixed-Size Pages?

1. **Alignment with OS pages** — the OS page cache operates in fixed-size units (usually 4 KB). Database pages that are multiples of OS pages transfer efficiently.
2. **Simple addressing** — page N is at byte offset `N × page_size`. No need for complex lookup structures.
3. **Buffer pool management** — fixed-size slots simplify memory allocation and replacement.
4. **Atomic writes** — some storage hardware can guarantee atomic writes at specific sizes.

### Common Page Sizes

| Database | Default Page Size | Configurable? |
|----------|-------------------|---------------|
| PostgreSQL | 8 KB | Compile-time only |
| MySQL InnoDB | 16 KB | Yes (4/8/16/32/64 KB) |
| SQLite | 4 KB | Yes (512 B – 64 KB) |
| SQL Server | 8 KB | No |
| Oracle | 8 KB | Yes (2/4/8/16/32 KB) |

---

## Heap Files

A **heap file** is the simplest way to organise pages: an unordered collection where new rows are appended wherever free space exists.

### How Rows Are Located

Each row has a physical address called a **tuple identifier (TID)** or **row ID (RID)**, composed of:

```
TID = (page_number, slot_number)

Example: (42, 3) → page 42, slot 3
```

### Free Space Management

The database needs to find pages with enough room for a new row. Common approaches:

```
Free Space Map (FSM)
┌────────┬────────────────┐
│ Page # │ Free Space (%) │
├────────┼────────────────┤
│   0    │    12%         │
│   1    │    45%         │  ← Insert here
│   2    │     0%         │
│   3    │    78%         │  ← Or here
│   4    │    90%         │
└────────┴────────────────┘
```

PostgreSQL maintains a **Free Space Map (FSM)** per table — a compact structure that tracks approximate free space in each page so inserts can quickly find a suitable page without scanning the entire file.

---

## Slotted Pages

Most relational databases use the **slotted page** layout. A page has a header at the top and data at the bottom, growing toward each other.

```
┌─────────────────────────────────────────────────┐
│                  Page Header                     │
│  (page_id, checksum, LSN, free_space_offset,    │
│   slot_count, flags)                             │
├─────────────────────────────────────────────────┤
│  Slot Array (grows →)                            │
│  [slot0: offset=8100, len=85]                   │
│  [slot1: offset=8015, len=92]                   │
│  [slot2: offset=7920, len=95]                   │
│  [slot3: ∅ (deleted)]                           │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
│            Free Space                            │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
│  Tuple 2 (95 bytes)  ← (grows ←)               │
│  Tuple 1 (92 bytes)                             │
│  Tuple 0 (85 bytes)                             │
└─────────────────────────────────────────────────┘
```

### Why Slotted?

- **Stable slot numbers** — external references (indexes) point to `(page, slot)`. When tuples are compacted or moved within the page, only the slot array is updated; indexes remain valid.
- **Variable-length tuples** — tuples of different sizes coexist naturally. The slot array stores each tuple's offset and length.
- **Deletion** — mark a slot as empty. Reclaim space during page compaction without affecting other slots.

### Page Header Fields

| Field | Purpose |
|-------|---------|
| `page_id` | Identifies this page within the file |
| `checksum` | Detects corruption (CRC32 or similar) |
| `LSN` | Log Sequence Number of last modification (used by WAL) |
| `free_space_offset` | Boundary between slot array and free space |
| `slot_count` | Number of slots (including deleted) |
| `flags` | Page type, full/empty hints |

---

## Tuple Structure

Each tuple (row) stored in a page has its own internal layout.

### PostgreSQL Tuple Layout

```
┌─────────────────────────────────────────────────┐
│              Tuple Header (23 bytes)             │
│  xmin (4B) │ xmax (4B) │ cid (4B) │ ctid (6B) │
│  infomask (2B) │ infomask2 (2B) │ hoff (1B)   │
├─────────────────────────────────────────────────┤
│              Null Bitmap (variable)              │
│  1 bit per column: 0 = NULL, 1 = present        │
├─────────────────────────────────────────────────┤
│              Column Data                         │
│  col1_value │ col2_value │ ... │ colN_value     │
│  (aligned to type-specific boundaries)           │
└─────────────────────────────────────────────────┘
```

| Header field | Purpose |
|-------------|---------|
| `xmin` | Transaction ID that inserted this tuple |
| `xmax` | Transaction ID that deleted/updated this tuple (0 if live) |
| `ctid` | Current tuple ID — points to the latest version if updated |
| `infomask` | Bit flags for visibility, locking, and tuple state |
| `null bitmap` | Tracks which columns are NULL without wasting storage |

The header overhead means very narrow tables (few small columns) can have significant per-row overhead relative to actual data.

---

## Row Store vs Column Store

### Row-Oriented (N-ary Storage Model, NSM)

All columns of a single row are stored contiguously.

```
Page contents (row store):
┌──────────────────────────────────────────────┐
│ Row1: [id=1, name="Alice", dept="Eng", sal=95000] │
│ Row2: [id=2, name="Bob",   dept="Mkt", sal=72000] │
│ Row3: [id=3, name="Carol", dept="Eng", sal=105000]│
└──────────────────────────────────────────────┘
```

**Strengths:** Reading/writing entire rows is a single I/O. Good for OLTP (point lookups, single-row inserts/updates).

### Column-Oriented (Decomposition Storage Model, DSM)

Each column is stored separately.

```
Column "id":     [1, 2, 3, 4, 5, ...]
Column "name":   ["Alice", "Bob", "Carol", ...]
Column "dept":   ["Eng", "Mkt", "Eng", ...]
Column "salary": [95000, 72000, 105000, ...]
```

**Strengths:** Analytical queries that read few columns out of many touch much less data. Columns of the same type compress extremely well.

### Comparison

| Property | Row Store | Column Store |
|----------|-----------|-------------|
| Point lookup by PK | Excellent (one page read) | Poor (must reconstruct from multiple columns) |
| Full-row insert | Single write | Write to every column file |
| Aggregate on 1 column | Reads entire rows | Reads only that column |
| Compression ratio | Moderate (mixed types per page) | Excellent (same type per page) |
| OLTP workloads | Preferred | Not suitable |
| OLAP / analytics | Inefficient | Preferred |
| Typical systems | PostgreSQL, MySQL, Oracle | ClickHouse, DuckDB, Redshift, Parquet |

### Hybrid Approaches

Some systems combine both. PostgreSQL's **columnar extensions** (Citus Columnar, Hydra) store cold analytical data in columnar format while keeping hot OLTP data in heap pages. Oracle's **In-Memory Column Store** keeps a columnar copy of selected tables in memory alongside the row-based storage on disk.

---

## Practical: Inspecting Page Layout in PostgreSQL

PostgreSQL exposes internal page structure via the `pageinspect` extension:

```sql
-- Enable the extension
CREATE EXTENSION pageinspect;

-- View page header
SELECT * FROM page_header(get_raw_page('my_table', 0));

-- View tuple headers
SELECT lp, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('my_table', 0));
```

Output reveals slot offsets, transaction IDs, and tuple lengths — making the slotted page model visible.

---

## Key Takeaways

- Databases read and write in fixed-size **pages**, not individual rows — page size (typically 4–16 KB) dictates I/O granularity.
- SSDs eliminated the seek penalty of HDDs but introduced write amplification; database designs still benefit from sequential access patterns.
- **Slotted pages** decouple external references (index pointers) from physical tuple placement within a page, enabling compaction without index updates.
- Every tuple carries a header with transaction metadata (`xmin`, `xmax`) that enables MVCC — this overhead is the cost of concurrent access.
- **Row stores** favour OLTP (whole-row access); **column stores** favour OLAP (aggregate queries on few columns). The workload determines which layout wins.
- Tools like `pageinspect` in PostgreSQL let you observe these internals directly.
