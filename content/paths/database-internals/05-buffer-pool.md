---
title: "Buffer Pool Management"
weight: 5
---

## The Memory–Disk Gap

Disk I/O (even on SSDs) is orders of magnitude slower than memory access. The **buffer pool** bridges this gap by caching frequently accessed disk pages in memory. A well-tuned buffer pool means most reads never touch disk.

```
Access latency comparison:
  L1 cache:     ~1 ns
  DRAM:         ~100 ns
  SSD random:   ~50,000 ns    (500× slower than RAM)
  HDD random:   ~10,000,000 ns (100,000× slower than RAM)
```

The buffer pool is the single most important performance structure in a database — it determines the **cache hit ratio**, which directly controls query latency.

---

## Buffer Pool Structure

The buffer pool is a fixed-size region of shared memory divided into **frames**, each the size of one database page.

```
┌───────────────────────────────────────────────────────┐
│                     Buffer Pool                        │
├──────────┬──────────┬──────────┬──────────┬───────────┤
│ Frame 0  │ Frame 1  │ Frame 2  │ Frame 3  │ Frame 4.. │
│ Page 42  │ Page 7   │ (empty)  │ Page 103 │ Page 15   │
│ dirty=Y  │ dirty=N  │          │ dirty=N  │ dirty=Y   │
│ pin=2    │ pin=0    │          │ pin=1    │ pin=0     │
│ ref=1    │ ref=0    │          │ ref=1    │ ref=0     │
└──────────┴──────────┴──────────┴──────────┴───────────┘

Page Table (hash map):
  page_id 42  → frame 0
  page_id 7   → frame 1
  page_id 103 → frame 3
  page_id 15  → frame 4
```

### Key Data Structures

| Structure | Purpose |
|-----------|---------|
| **Frame** | Fixed-size memory slot holding one page's contents |
| **Page table** | Hash map from page ID → frame number (not the OS page table) |
| **Dirty flag** | Indicates page has been modified in memory but not written to disk |
| **Pin count** | Number of active references; page cannot be evicted while pinned |
| **Reference bit** | Used by clock-based eviction to track recent access |

---

## Page Access Flow

When a query needs a page:

```
function get_page(page_id):
    1. Look up page_id in page table
    
    2. If found (CACHE HIT):
       a. Increment pin count
       b. Set reference bit
       c. Return pointer to frame
    
    3. If not found (CACHE MISS):
       a. Find a victim frame (eviction policy)
       b. If victim is dirty:
          - Flush victim page to disk (write-back)
       c. Read requested page from disk into frame
       d. Update page table: page_id → new frame
       e. Set pin count = 1, dirty = false
       f. Return pointer to frame

function release_page(page_id, modified):
    1. Decrement pin count
    2. If modified: set dirty flag
```

---

## Eviction Policies

When the buffer pool is full and a new page is needed, a **victim** frame must be chosen. Only frames with `pin_count = 0` are candidates.

### LRU (Least Recently Used)

Evict the page that was accessed longest ago.

```
Access sequence: A B C D A B E

LRU list (most recent → least recent):
  After A B C D:   [D, C, B, A]
  After A:         [A, D, C, B]
  After B:         [B, A, D, C]
  After E:         [E, B, A, D]  ← C evicted
```

| Strength | Weakness |
|----------|----------|
| Good for recency-based access | **Sequential flood**: a full table scan evicts all frequently used pages |
| Simple to understand | Maintaining an exact LRU list is expensive (O(1) with doubly-linked list + hash map, but high concurrency overhead) |

### LRU-K

Track the last *K* access timestamps. Evict the page whose *K*-th most recent access is oldest. LRU-2 is common.

This resists sequential floods because a single scan touches each page only once, so scanned pages never build up a second access timestamp.

### Clock (Second Chance)

An approximation of LRU that avoids maintaining a sorted list. Used by PostgreSQL.

```
Buffer frames arranged in a circle:

        ┌───┐
    ┌───│ A │───┐
    │   │ref=1│  │
    │   └───┘   │
  ┌───┐      ┌───┐
  │ D │      │ B │    Clock hand sweeps clockwise
  │ref=0│    │ref=1│
  └───┘      └───┘
    │   ┌───┐   │
    └───│ C │───┘
        │ref=1│
        └───┘
            ↑ hand

Algorithm:
  while frame[hand].ref == 1:
      frame[hand].ref = 0     ← Give a second chance
      hand = (hand + 1) % N
  
  evict frame[hand]           ← ref was already 0
  hand = (hand + 1) % N
```

| Property | LRU | Clock |
|----------|-----|-------|
| Accuracy | Exact ordering | Approximate |
| Overhead per access | Update linked list (contention) | Set one bit (cheap) |
| Concurrency | Lock-heavy | Lightweight |
| Used by | MySQL InnoDB (LRU variant) | PostgreSQL |

### PostgreSQL's Clock Sweep

PostgreSQL uses a clock sweep with a `usage_count` (0–5) instead of a single reference bit. Each access increments the counter (capped at 5). The clock hand decrements the counter by 1; a page is evicted when its counter reaches 0.

This means heavily used pages survive multiple sweeps, while occasionally accessed pages are evicted faster — effectively an approximation of LFU (Least Frequently Used).

---

## Dirty Page Management

A **dirty page** has been modified in the buffer pool but not yet written to disk. Dirty pages must be flushed before eviction, adding latency to the eviction process.

### Write-Back Strategy

Databases flush dirty pages in the background to avoid stalling the eviction process:

```
Background Writer (PostgreSQL: bgwriter)
  Periodically scans buffer pool
  Flushes dirty pages with low usage_count
  Goal: ensure some clean frames are always available

Checkpointer
  Periodically flushes ALL dirty pages
  Writes a checkpoint record to WAL
  Enables WAL truncation
```

### Flush Order Constraint

The **WAL protocol** constrains flushing: a dirty page cannot be written to disk until all WAL records for changes to that page have been flushed first. The page's `page_lsn` must be ≤ the flushed WAL position.

```
Page 42 in buffer pool:
  page_lsn = 10050
  
WAL flushed up to LSN 10048

Cannot flush page 42 yet — must wait until WAL is flushed to ≥ 10050.
```

---

## Pin and Unpin

**Pinning** prevents a page from being evicted while it is actively being read or modified.

```
Transaction T1:                    Buffer Pool:
  pin(page 42)                     page 42: pin_count = 1
  read row from page 42
  pin(page 42) again               page 42: pin_count = 2  (nested)
  modify row on page 42
  unpin(page 42)                   page 42: pin_count = 1
  unpin(page 42)                   page 42: pin_count = 0  ← now evictable
```

A page with `pin_count > 0` is never chosen as an eviction victim. If all frames are pinned, the system must wait — this is a potential deadlock source if not carefully managed.

---

## Prefetching

Instead of waiting for a cache miss, the buffer pool can **prefetch** pages it expects to need soon.

### Sequential Prefetch

```
Query: SELECT * FROM large_table  (sequential scan)

Access pattern: page 1, 2, 3, 4, 5, ...

After detecting sequential access (pages 1, 2, 3):
  Prefetch pages 4, 5, 6, 7 asynchronously
  By the time query needs page 4, it is already in memory
```

### Index Prefetch

When an index scan produces a list of heap TIDs, the database can sort them by page number and prefetch those heap pages.

```
Index scan returns TIDs: (42,1), (42,3), (7,2), (103,5), (7,4)

Sort by page: 7, 7, 42, 42, 103
Prefetch pages 7, 42, 103 simultaneously
```

PostgreSQL calls this **bitmap heap scan** — the bitmap index scan collects TIDs, sorts them, and fetches heap pages in physical order.

---

## Buffer Pool vs OS Page Cache

The operating system also caches file contents in its **page cache**. This creates a potential double-caching problem.

```
┌──────────────────────┐
│  Database Process     │
│  ┌─────────────────┐ │
│  │  Buffer Pool     │ │  ← Database-managed cache
│  │  (shared_buffers)│ │
│  └────────┬────────┘ │
└───────────┼──────────┘
            │ read()/write()
┌───────────┼──────────┐
│  OS Kernel│          │
│  ┌────────▼────────┐ │
│  │  Page Cache      │ │  ← OS-managed cache
│  └────────┬────────┘ │
└───────────┼──────────┘
            │ disk I/O
      ┌─────▼─────┐
      │   Disk     │
      └───────────┘
```

### Why Not Just Use the OS Cache?

| Feature | Buffer Pool | OS Page Cache |
|---------|-------------|---------------|
| Eviction policy | Domain-specific (clock sweep with usage count) | Generic LRU |
| Pin/unpin | Yes (prevents eviction during use) | No |
| Dirty page control | Precise (WAL-aware flushing) | OS decides when to flush |
| Prefetch control | Database-directed | OS heuristics |
| Memory accounting | Explicit, configurable | Opaque, shared with all processes |

Databases use the buffer pool for precise control. However, most also benefit from the OS page cache as a second-level cache — PostgreSQL recommends setting `shared_buffers` to 25% of RAM, leaving the rest for the OS page cache.

### Direct I/O

Some databases bypass the OS page cache entirely using **O_DIRECT** (Linux) or equivalent:

| Approach | Used by | Rationale |
|----------|---------|-----------|
| Buffered I/O + OS cache | PostgreSQL | Simpler, OS cache acts as second level |
| Direct I/O | MySQL InnoDB, Oracle | Avoid double caching, full control |

MySQL InnoDB defaults to `innodb_flush_method = O_DIRECT` to prevent double caching.

---

## Sizing the Buffer Pool

| Database | Parameter | Guideline |
|----------|-----------|-----------|
| PostgreSQL | `shared_buffers` | 25% of system RAM (OS cache handles the rest) |
| MySQL InnoDB | `innodb_buffer_pool_size` | 70–80% of system RAM (uses direct I/O) |
| Oracle | `SGA_TARGET` | 60–75% of system RAM |

The **buffer cache hit ratio** measures effectiveness:

```
Hit ratio = buffer_hits / (buffer_hits + disk_reads)

Target: > 99% for OLTP workloads
```

```sql
-- PostgreSQL: check hit ratio
SELECT
  sum(blks_hit) AS hits,
  sum(blks_read) AS reads,
  round(sum(blks_hit) / (sum(blks_hit) + sum(blks_read))::numeric, 4) AS ratio
FROM pg_stat_database;
```

---

## Key Takeaways

- The buffer pool is a fixed-size memory region that caches disk pages — it is the primary determinant of database I/O performance.
- **Clock sweep** (PostgreSQL) approximates LRU with lower concurrency overhead. MySQL InnoDB uses a modified LRU with a midpoint to resist sequential flood.
- Dirty pages are flushed asynchronously by background writers; the WAL protocol constrains flush order.
- **Pinning** protects pages from eviction during active use; all eviction candidates must have `pin_count = 0`.
- **Prefetching** (sequential and index-based) hides I/O latency by reading pages before they are needed.
- The database buffer pool and OS page cache serve different purposes — the buffer pool provides WAL-aware, domain-specific caching that the OS cannot replicate.
- Buffer pool sizing depends on whether the database uses direct I/O (allocate most of RAM) or buffered I/O (allocate ~25%, let the OS cache the rest).
