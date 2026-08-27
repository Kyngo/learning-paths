---
title: "Write-Ahead Logging"
weight: 4
---

## The Durability Problem

A database crash can happen at any moment — power failure, kernel panic, hardware fault. If modified pages exist only in the buffer pool (memory), they are lost. Writing every modified page to disk immediately would be safe but catastrophically slow (random I/O for every change).

**Write-Ahead Logging (WAL)** solves this: before modifying any data page, write a log record describing the change to a sequential log file. If the system crashes, replay the log to reconstruct any modifications that were not yet flushed to the data files.

---

## The WAL Protocol

The fundamental rule, also called the **write-ahead log protocol**:

> A data page must not be flushed to disk until all log records describing changes to that page have been flushed to the WAL.

```
Transaction T1: UPDATE accounts SET balance = 500 WHERE id = 7

Timeline:
  1. Write log record to WAL buffer   ← "page 42, offset 120: old=300, new=500"
  2. Modify page 42 in buffer pool    ← In-memory change (dirty page)
  3. Flush WAL buffer to disk          ← WAL is now durable
  4. Acknowledge commit to client      ← Transaction is "committed"
  5. ... later: flush page 42 to disk  ← Happens asynchronously
```

If the system crashes between steps 4 and 5, the WAL contains the record. Recovery replays it.

---

## Log Record Structure

Each log record contains enough information to both **redo** (reapply) and **undo** (reverse) a change.

```
┌──────────────────────────────────────────────────┐
│               WAL Log Record                      │
├──────────────────────────────────────────────────┤
│  LSN: 10042                                      │
│  Transaction ID: 73                              │
│  Type: UPDATE                                    │
│  Page ID: 42                                     │
│  Offset: 120                                     │
│  Before Image: 0x0000012C  (old value: 300)     │
│  After Image:  0x000001F4  (new value: 500)     │
│  Prev LSN: 10038  (previous record for txn 73)  │
└──────────────────────────────────────────────────┘
```

### Record Types

| Type | Purpose |
|------|---------|
| **UPDATE** | Data modification (before + after images) |
| **INSERT** | New tuple added (after image only) |
| **DELETE** | Tuple removed (before image only) |
| **COMMIT** | Transaction committed |
| **ABORT** | Transaction rolled back |
| **CHECKPOINT** | Snapshot of system state for recovery |
| **CLR** (Compensation Log Record) | Undo action during recovery |

---

## Log Sequence Numbers (LSN)

Every log record receives a monotonically increasing **Log Sequence Number (LSN)**. LSNs are the backbone of crash recovery — they establish a total ordering of all modifications.

```
WAL:
  LSN 100: T1 inserts row into page 5
  LSN 101: T2 updates row in page 12
  LSN 102: T1 updates row in page 5
  LSN 103: T2 commits
  LSN 104: T1 commits
  LSN 105: T3 inserts row into page 8
```

Every data page also stores the LSN of the **last modification applied to it**:

```
Page 5 on disk:
  page_lsn = 100

Page 5 in buffer pool:
  page_lsn = 102  (two changes applied in memory)
```

During recovery, the system compares `page_lsn` on disk with the WAL. If the WAL contains records with LSN > `page_lsn` for that page, those changes need to be redone.

---

## ARIES Crash Recovery

**ARIES** (Algorithm for Recovery and Isolation Exploiting Semantics) is the standard recovery algorithm used by most databases (IBM DB2, SQL Server, PostgreSQL's recovery is ARIES-inspired).

Recovery has three phases:

```
                   Crash
                     │
    ┌────────────────┴────────────────┐
    │         WAL on disk              │
    │  ... [100] [101] [102] [103] ...│
    └─────────────────────────────────┘
    
    Recovery:
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ 1. ANALYSIS│ → │ 2. REDO   │ → │ 3. UNDO   │
    └──────────┘   └──────────┘   └──────────┘
```

### Phase 1: Analysis

Scan the WAL forward from the last checkpoint to determine:
- Which transactions were **active** (uncommitted) at crash time
- Which pages are **dirty** (modified but not yet flushed to disk)

```
Analysis output:
  Active transactions: {T3, T5}        ← Need to be undone
  Dirty page table:    {page5: LSN=100, page12: LSN=101}
```

### Phase 2: Redo

Scan the WAL forward again, reapplying **all** logged changes to bring pages to their most recent state — even changes from uncommitted transactions.

```
For each log record with LSN > page_lsn on disk:
    Apply the after-image to the page
    Update page_lsn
```

After redo, the database is in the exact state it was at the moment of the crash — including uncommitted changes.

### Phase 3: Undo

Roll back all uncommitted transactions by applying their changes in reverse, using the before-images from log records.

```
For each active (uncommitted) transaction:
    Follow prev_lsn chain backward
    For each record:
        Apply before-image (reverse the change)
        Write a CLR (Compensation Log Record)
```

CLRs ensure that if the system crashes *during recovery*, the undo work is not repeated.

---

## Checkpointing

Without checkpoints, recovery would need to replay the entire WAL from the beginning of time. Checkpoints establish a known-good state that limits recovery scope.

### Types of Checkpoints

| Type | Method | Downtime |
|------|--------|----------|
| **Sharp checkpoint** | Flush all dirty pages, block all writes | High (simple, rarely used) |
| **Fuzzy checkpoint** | Record dirty page table and active transactions without flushing | None (used in practice) |

```
Fuzzy Checkpoint:
┌──────────────────────────────────────────┐
│  CHECKPOINT record in WAL                │
│  Active transactions: {T3, T5}           │
│  Dirty page table:                       │
│    page 5:  oldest_lsn = 10040          │
│    page 12: oldest_lsn = 10055          │
│    page 8:  oldest_lsn = 10061          │
└──────────────────────────────────────────┘
```

Recovery starts from the checkpoint's dirty page table rather than the beginning of the WAL.

### Checkpoint Frequency Trade-Off

| Frequent checkpoints | Infrequent checkpoints |
|---------------------|----------------------|
| Faster recovery (less WAL to replay) | Slower recovery |
| More I/O during normal operation | Less overhead |
| WAL can be truncated sooner | WAL grows larger |

PostgreSQL defaults to checkpointing every 5 minutes or after 1 GB of WAL written (`checkpoint_timeout`, `max_wal_size`).

---

## Physiological Logging

Most databases use **physiological logging** — a hybrid between physical and logical logging:

| Logging type | Granularity | Example |
|-------------|-------------|---------|
| **Physical** | Exact byte changes | "byte 120 of page 42: change 0x12C → 0x1F4" |
| **Logical** | High-level operation | "UPDATE accounts SET balance=500 WHERE id=7" |
| **Physiological** | Physical page + logical within page | "On page 42: update slot 3, set column 2 to 500" |

Physiological logging is the sweet spot:
- **Physical enough** to identify the exact page (no need to re-execute queries)
- **Logical enough** within the page to handle tuple movement during compaction
- **Idempotent** — applying the same log record twice produces the same result

---

## Group Commit

Flushing the WAL to disk (calling `fsync`) for every single transaction is expensive. **Group commit** batches multiple transactions' WAL records into a single flush.

```
Without group commit:
  T1 commit → fsync WAL → ack T1
  T2 commit → fsync WAL → ack T2
  T3 commit → fsync WAL → ack T3
  3 fsyncs (each ~0.5–2 ms on SSD)

With group commit:
  T1 commit → queue
  T2 commit → queue
  T3 commit → queue
  → Single fsync for all three → ack T1, T2, T3
  1 fsync total
```

### Trade-off

| Setting | Latency | Throughput |
|---------|---------|-----------|
| `fsync` every commit | Lowest latency per txn | Lower throughput (limited by fsync speed) |
| Group commit (small delay) | Slightly higher latency | Much higher throughput |
| No `fsync` (dangerous) | Lowest latency | Risk of data loss on crash |

PostgreSQL controls this with `commit_delay` (microseconds to wait for additional commits before flushing) and `commit_siblings` (minimum concurrent transactions before delay applies).

---

## WAL in Practice

### PostgreSQL WAL

```
$PGDATA/pg_wal/
├── 000000010000000000000001   (16 MB segment)
├── 000000010000000000000002
├── 000000010000000000000003
└── ...
```

Each segment is 16 MB by default. PostgreSQL recycles old segments rather than deleting them. Replication and point-in-time recovery work by shipping and replaying WAL segments.

### MySQL InnoDB Redo Log

MySQL InnoDB uses a circular redo log:

```
┌───────────┐    ┌───────────┐
│ ib_redo_0 │ ── │ ib_redo_1 │ ── (wraps around)
└───────────┘    └───────────┘
     ▲                         
     │ write_pos               
     │ checkpoint_pos          
```

The distance between `checkpoint_pos` and `write_pos` determines how much WAL must be replayed on recovery.

---

## Key Takeaways

- The WAL protocol guarantees durability without requiring immediate data page flushes — log records must hit disk before modified pages do.
- Every log record has an **LSN** that establishes a total order of modifications, enabling precise crash recovery.
- **ARIES** recovery follows three phases: analysis (find dirty pages and active transactions), redo (replay all changes), undo (reverse uncommitted work).
- **Checkpoints** bound recovery time by recording a snapshot of system state; fuzzy checkpoints avoid blocking normal operations.
- **Physiological logging** (physical page + logical content) is the standard — it is idempotent and handles page compaction gracefully.
- **Group commit** amortises the cost of `fsync` across multiple transactions, trading a tiny latency increase for major throughput gains.
- The WAL is also the foundation for replication — streaming WAL records to replicas is how PostgreSQL, MySQL, and others implement physical replication.
