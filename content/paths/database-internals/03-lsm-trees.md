---
title: "LSM Trees & Log-Structured Storage"
weight: 3
---

## The Write Optimisation Problem

B+trees provide excellent read performance, but writes are expensive: inserting a single key may require reading a leaf page, modifying it, and writing it back — all random I/O. For write-heavy workloads (time-series, event logs, messaging), this becomes a bottleneck.

**Log-Structured Merge-trees (LSM-trees)** flip the trade-off: absorb all writes in memory, then flush sorted runs to disk sequentially. Reads become more expensive, but writes are dramatically cheaper.

---

## LSM-Tree Architecture

```
                     Write Path
                        │
                        ▼
            ┌───────────────────┐
            │    Memtable       │  ← In-memory sorted structure
            │  (Red-Black Tree  │     (e.g., skip list, AVL tree)
            │   or Skip List)   │
            └────────┬──────────┘
                     │ flush (when full)
                     ▼
    Disk:  ┌─────────────────────────────────────┐
           │  Level 0:  SST-A  SST-B  SST-C     │  ← Recently flushed
           │  Level 1:  SST-D  SST-E             │  ← Compacted
           │  Level 2:  SST-F                    │  ← Compacted further
           └─────────────────────────────────────┘
              Each SSTable is sorted and immutable
```

### Core Components

| Component | Description |
|-----------|-------------|
| **Memtable** | In-memory sorted data structure. All writes go here first. |
| **WAL** | Write-ahead log for durability. Written before memtable insertion. |
| **SSTable** | Sorted String Table — an immutable, sorted file on disk. |
| **Levels** | Organisational tiers on disk. Lower levels hold newer, smaller SSTables. |
| **Compaction** | Background process that merges SSTables to reduce read overhead. |

---

## Write Path

Every write follows the same sequence:

```
function put(key, value):
    1. Append (key, value) to WAL          ← Sequential disk write (durable)
    2. Insert (key, value) into memtable   ← In-memory (fast)
    3. If memtable exceeds size threshold:
       a. Freeze current memtable
       b. Create new empty memtable
       c. Flush frozen memtable to disk as new SSTable in Level 0
       d. Truncate WAL entries for flushed data
```

**Deletes** are handled by writing a **tombstone** — a special marker that says "this key is deleted." The actual data is removed during compaction.

```
Write:   PUT(key="user:1", value="Alice")   → stored in memtable
Delete:  DELETE(key="user:1")                → tombstone in memtable
Update:  PUT(key="user:1", value="Bob")      → new entry in memtable
```

---

## SSTable Format

An SSTable is a sorted, immutable file with the following structure:

```
┌──────────────────────────────────────────┐
│              SSTable File                 │
├──────────────────────────────────────────┤
│  Data Block 0                            │
│    key1=val1 │ key2=val2 │ key3=val3     │
├──────────────────────────────────────────┤
│  Data Block 1                            │
│    key4=val4 │ key5=val5 │ key6=val6     │
├──────────────────────────────────────────┤
│  ...                                     │
├──────────────────────────────────────────┤
│  Index Block                             │
│    block0 → offset 0                     │
│    block1 → offset 4096                  │
├──────────────────────────────────────────┤
│  Bloom Filter                            │
├──────────────────────────────────────────┤
│  Footer (metadata, offsets, checksums)   │
└──────────────────────────────────────────┘
```

Key properties:
- **Sorted** — enables binary search and efficient merging
- **Immutable** — once written, never modified (simplifies concurrency)
- **Block-compressed** — data blocks are individually compressed (LZ4, Snappy, Zstd)

---

## Read Path

Reading from an LSM-tree searches from newest to oldest:

```
function get(key):
    1. Search memtable                     ← Fastest (in memory)
    2. Search immutable memtable (if any)  ← Being flushed
    3. For each level, newest SSTable first:
       a. Check bloom filter              ← Skip if key definitely absent
       b. Search index block              ← Find correct data block
       c. Search data block               ← Binary search within block
    4. Return first match found
       (or NOT_FOUND if all levels exhausted)
```

### Read Amplification

A point lookup might check:
- 1 memtable
- L0: 3 SSTables (overlapping)
- L1: 1 SSTable
- L2: 1 SSTable

That is 6 lookups for a single key. This is **read amplification** — the price of write optimisation.

---

## Bloom Filters

Bloom filters dramatically reduce read amplification by quickly ruling out SSTables that *definitely* do not contain a key.

```
Bloom Filter (probabilistic set membership):
┌──────────────────────────────────────┐
│  Bit array: [0,1,0,1,1,0,0,1,0,1]  │
└──────────────────────────────────────┘

Insert key "user:42":
  hash1("user:42") → bit 1   (set to 1)
  hash2("user:42") → bit 3   (set to 1)
  hash3("user:42") → bit 7   (set to 1)

Query key "user:99":
  hash1("user:99") → bit 2   (is 0)  → DEFINITELY NOT HERE
  Skip this SSTable entirely.

Query key "user:42":
  hash1 → bit 1 (1), hash2 → bit 3 (1), hash3 → bit 7 (1)
  → MAYBE HERE (could be false positive)
  → Read the SSTable to confirm.
```

| Bits per key | False positive rate |
|-------------|---------------------|
| 8 | ~2.2% |
| 10 | ~1.0% |
| 12 | ~0.5% |
| 16 | ~0.02% |

At 10 bits per key, bloom filters eliminate ~99% of unnecessary SSTable reads.

---

## Compaction Strategies

Compaction merges SSTables to: (1) reduce the number of files reads must check, (2) remove tombstones and obsolete versions, and (3) reclaim disk space.

### Size-Tiered Compaction (STCS)

```
Level 0:  [A] [B] [C] [D]    ← 4 similarly-sized SSTables
                │
                ▼  merge all 4
Level 1:  [    ABCD    ]     ← 1 larger SSTable

When Level 1 accumulates enough files:
Level 1:  [ABCD] [EFGH] [IJKL] [MNOP]
                    │
                    ▼  merge
Level 2:  [      ABCDEFGHIJKLMNOP      ]
```

| Property | Value |
|----------|-------|
| Trigger | When a level accumulates N similarly-sized SSTables |
| Write amplification | Lower (~O(N × size_ratio)) |
| Read amplification | Higher (overlapping keys across files in same level) |
| Space amplification | Higher (multiple copies of same key exist until compacted) |
| Best for | Write-heavy workloads (Cassandra default) |

### Leveled Compaction (LCS)

```
Level 0:  [A] [B]              ← Flushed from memtable (may overlap)
Level 1:  [C] [D] [E]         ← Non-overlapping, sorted runs
Level 2:  [F] [G] [H] [I] [J]  ← 10× larger than L1, non-overlapping
Level 3:  [.........]          ← 10× larger than L2
```

Each level (except L0) maintains the invariant that SSTables within a level have **non-overlapping key ranges**. Compaction picks one SSTable from level *L* and merges it with overlapping SSTables in level *L+1*.

| Property | Value |
|----------|-------|
| Trigger | When a level exceeds its size limit |
| Write amplification | Higher (~O(size_ratio × levels)) |
| Read amplification | Lower (at most 1 SSTable per level for a point lookup) |
| Space amplification | Lower (less duplication) |
| Best for | Read-heavy workloads (RocksDB default, LevelDB) |

### Comparison

| Metric | Size-Tiered | Leveled |
|--------|-------------|---------|
| Write amplification | Low (2–5×) | High (10–30×) |
| Read amplification | High | Low |
| Space amplification | High (up to 2×) | Low (~1.1×) |
| Compaction I/O | Bursty (large merges) | Steady (incremental merges) |
| SSD wear | Lower | Higher |

---

## Amplification Trade-Offs

Every LSM-tree design navigates three competing factors:

```
         Write Amp
            ▲
           / \
          /   \
         /     \
        /       \
       ▼─────────▼
  Read Amp    Space Amp

You can optimise for two at the expense of the third.
```

| Strategy | Optimises | Sacrifices |
|----------|----------|-----------|
| Size-tiered | Write amp, space amp (somewhat) | Read amp |
| Leveled | Read amp, space amp | Write amp |
| FIFO (time-series) | Write amp | Read amp, space amp (TTL-based cleanup) |
| Universal (RocksDB) | Configurable blend | Depends on tuning |

---

## LSM-Tree vs B+Tree

| Property | B+Tree | LSM-Tree |
|----------|--------|----------|
| Write pattern | Random (in-place update) | Sequential (append + compact) |
| Read pattern | Predictable (tree height) | Variable (depends on compaction state) |
| Write throughput | Lower | Higher (especially bulk writes) |
| Read latency | Lower, consistent | Higher, less predictable |
| Space usage | Low overhead | Higher (tombstones, duplicates until compaction) |
| Range scans | Excellent (leaf chain) | Good (merged iterators across levels) |
| Concurrency | Requires latching | Simpler (immutable SSTables) |
| Typical use | OLTP (PostgreSQL, MySQL) | Write-heavy (Cassandra, RocksDB, HBase) |

---

## Real-World Implementations

| System | Storage Engine | Compaction Default |
|--------|---------------|-------------------|
| LevelDB | LSM | Leveled |
| RocksDB | LSM | Leveled (with universal option) |
| Cassandra | LSM | Size-tiered (LCS optional) |
| HBase | LSM | Size-tiered |
| ScyllaDB | LSM | Size-tiered / incremental |
| CockroachDB | RocksDB/Pebble (LSM) | Leveled |
| TiKV (TiDB) | RocksDB (LSM) | Leveled |
| SQLite | B-tree | N/A |
| PostgreSQL | B+tree (heap) | N/A |
| MySQL InnoDB | B+tree | N/A |

---

## Key Takeaways

- LSM-trees trade read performance for dramatically better write throughput by converting random writes into sequential I/O.
- The **memtable** absorbs writes in memory; **SSTables** are sorted, immutable files flushed to disk.
- **Bloom filters** are essential — they let reads skip SSTables that definitely don't contain the target key, reducing read amplification by ~99%.
- **Size-tiered compaction** favours write throughput; **leveled compaction** favours read performance. Choose based on your workload.
- Write amplification, read amplification, and space amplification are the three axes of LSM-tree tuning — optimising one worsens another.
- Deletes use **tombstones**, not immediate removal. Data is physically reclaimed only during compaction.
- Most modern distributed databases (CockroachDB, TiDB, Cassandra) use LSM-trees because distributed writes map well to the append-only model.
