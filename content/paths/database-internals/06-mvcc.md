---
title: "MVCC Implementation"
weight: 6
---

## Why MVCC?

Without concurrency control, simultaneous readers and writers would see inconsistent data — partial updates, phantom rows, or torn reads. Lock-based approaches (shared/exclusive locks on every row) are correct but cause readers to block writers and vice versa.

**Multi-Version Concurrency Control (MVCC)** solves this by keeping multiple versions of each row. Readers see a consistent snapshot without acquiring locks; writers create new versions instead of overwriting in place. The result: **readers never block writers, and writers never block readers**.

The Databases & SQL path covered isolation levels from the user's perspective. This section covers how MVCC is implemented inside the engine.

---

## MVCC Approaches

| Approach | How it works | Used by |
|----------|-------------|---------|
| **Append-only** (multi-version in heap) | Old and new versions coexist in the same table | PostgreSQL |
| **Undo log** (in-place update + rollback segment) | Latest version in table; old versions in separate undo space | MySQL InnoDB, Oracle |
| **Delta storage** | Store only the diff between versions | SQL Server (row versioning in tempdb) |

---

## PostgreSQL's MVCC: Tuple Versioning

PostgreSQL stores all versions of a row directly in the heap table. Each tuple has header fields that control visibility.

### Tuple Header Fields

```
┌──────────────────────────────────────────────────┐
│  xmin = 100    │  Transaction that created this   │
│  xmax = 0      │  Transaction that deleted this   │
│                 │  (0 = still live)                │
│  ctid = (5,3)  │  Pointer to current tuple version│
│  infomask      │  Bit flags for commit status     │
└──────────────────────────────────────────────────┘
```

### INSERT

```
Transaction 100: INSERT INTO users VALUES (1, 'Alice')

Heap page:
┌─────────────────────────────────┐
│  Tuple A                        │
│  xmin=100, xmax=0, data='Alice' │  ← Visible to txn 100 and later
└─────────────────────────────────┘
```

### UPDATE (creates a new version)

PostgreSQL does not update in place. An UPDATE is a DELETE of the old version + INSERT of a new version.

```
Transaction 200: UPDATE users SET name='Bob' WHERE id=1

Heap page:
┌──────────────────────────────────────┐
│  Tuple A (old version)               │
│  xmin=100, xmax=200, ctid→(5,7)     │  ← Marked as deleted by txn 200
├──────────────────────────────────────┤
│  Tuple B (new version)               │
│  xmin=200, xmax=0, data='Bob'       │  ← Created by txn 200
└──────────────────────────────────────┘
```

### DELETE

```
Transaction 300: DELETE FROM users WHERE id=1

Heap page:
┌──────────────────────────────────────┐
│  Tuple B                             │
│  xmin=200, xmax=300                  │  ← Marked as deleted by txn 300
└──────────────────────────────────────┘
```

No data is physically removed. The tuple is invisible to new transactions but still occupies space until vacuum.

---

## Visibility Rules

When a transaction reads a tuple, it must decide: is this version visible to me?

### Snapshot

Each transaction (at `REPEATABLE READ` or `SERIALIZABLE` isolation) takes a **snapshot** at its start:

```
Snapshot for transaction 250:
  xmin = 250        (my transaction ID)
  xmax = 260        (next unassigned txn ID)
  active = [255, 258]  (in-progress transactions)
```

### Visibility Check

```
function is_visible(tuple, snapshot):
    // Check xmin (inserter)
    if tuple.xmin is not committed:
        if tuple.xmin == snapshot.my_txn:
            // I inserted this — visible (unless I also deleted it)
            return tuple.xmax == 0 or !is_committed(tuple.xmax)
        else:
            return false  // Uncommitted insert by another txn
    
    if tuple.xmin >= snapshot.xmax:
        return false  // Inserted after my snapshot
    
    if tuple.xmin in snapshot.active:
        return false  // Inserted by a concurrent txn still active at snapshot time
    
    // xmin is committed and visible. Check xmax (deleter)
    if tuple.xmax == 0:
        return true  // Not deleted
    
    if tuple.xmax is not committed:
        return true  // Delete not yet committed — still visible
    
    if tuple.xmax >= snapshot.xmax:
        return true  // Deleted after my snapshot
    
    if tuple.xmax in snapshot.active:
        return true  // Deleted by concurrent txn active at snapshot time
    
    return false  // Deleted and committed before my snapshot
```

### Hint Bits

Checking whether a transaction is committed requires looking up the **CLOG** (commit log). To avoid repeated lookups, PostgreSQL sets **hint bits** in the tuple header after the first visibility check:

```
infomask flags:
  HEAP_XMIN_COMMITTED   → xmin is known committed
  HEAP_XMIN_INVALID     → xmin is known aborted
  HEAP_XMAX_COMMITTED   → xmax is known committed
  HEAP_XMAX_INVALID     → xmax is known aborted
```

Once set, subsequent visibility checks skip the CLOG lookup entirely.

---

## Snapshot Isolation Implementation

### READ COMMITTED

Each **statement** gets a fresh snapshot. Different statements in the same transaction may see different data.

```
Transaction T1 (READ COMMITTED):
  Statement 1: snapshot at LSN 100  → sees data as of LSN 100
  -- T2 commits a change --
  Statement 2: snapshot at LSN 150  → sees T2's changes
```

### REPEATABLE READ

One snapshot for the **entire transaction**. All statements see the same consistent state.

```
Transaction T1 (REPEATABLE READ):
  BEGIN → snapshot at LSN 100
  Statement 1: sees data as of LSN 100
  -- T2 commits a change --
  Statement 2: still sees data as of LSN 100  ← T2's changes invisible
```

### Write Conflicts

Under `REPEATABLE READ`, if two transactions try to update the same row, PostgreSQL detects the conflict:

```
T1: UPDATE users SET name='X' WHERE id=1   ← acquires row lock
T2: UPDATE users SET name='Y' WHERE id=1   ← blocked, waits for T1

If T1 commits:
  T2 gets ERROR: could not serialize access
  (because T2's snapshot no longer matches reality)
```

---

## Vacuum and Garbage Collection

Dead tuples (old versions no longer visible to any transaction) waste space. PostgreSQL's **VACUUM** reclaims this space.

### Why Vacuum Is Necessary

```
Without vacuum:
  Table size grows continuously even if row count is stable
  Index entries point to dead tuples (index bloat)
  Transaction ID wraparound risk (see below)
```

### How Vacuum Works

```
function vacuum(table):
    oldest_active_snapshot = get_oldest_active_snapshot()
    
    for each page in table:
        for each tuple in page:
            if tuple is dead (xmax committed, invisible to all snapshots):
                mark tuple's space as reusable
                remove index entries pointing to dead tuple
        
        update Free Space Map (FSM)
```

### Autovacuum

PostgreSQL runs vacuum automatically in the background:

```
Autovacuum triggers when:
  dead_tuples > autovacuum_vacuum_threshold 
                + autovacuum_vacuum_scale_factor × table_size

Defaults:
  threshold = 50 tuples
  scale_factor = 0.2 (20% of table must be dead)
```

### VACUUM vs VACUUM FULL

| Operation | What it does | Locks | Reclaims disk space? |
|-----------|-------------|-------|---------------------|
| `VACUUM` | Marks dead tuples as reusable | No exclusive lock | No (space reused internally) |
| `VACUUM FULL` | Rewrites entire table, compacting | Exclusive lock (blocks all access) | Yes (shrinks file on disk) |

---

## Visibility Map

The **visibility map** is a bitmap with one bit per heap page. If the bit is set, the page contains only tuples visible to all current and future transactions.

```
Visibility Map:
  Page 0: 1  ← All tuples visible (skip during vacuum and index-only scans)
  Page 1: 0  ← Has dead or recently inserted tuples
  Page 2: 1
  Page 3: 0
  ...
```

Benefits:
- **Vacuum** skips pages with the bit set — no dead tuples to clean
- **Index-only scans** can return results from the index without visiting the heap page (if the visibility bit is set)

---

## Transaction ID Wraparound

PostgreSQL uses 32-bit transaction IDs (XIDs), which wrap around after ~4 billion transactions.

```
XID space (circular):

        0
        │
   2B ──┤──── 2B
        │
  "future" │ "past"
        │
```

Every XID more than 2 billion transactions in the past is considered "in the future" after wraparound — making old, committed data suddenly invisible.

### Prevention: Freeze

Vacuum marks very old tuples as **frozen** — replacing their `xmin` with a special value (`FrozenTransactionId = 2`) that is always considered committed and visible, regardless of XID comparison.

```
Before freeze: xmin = 100000 (old, risk of wraparound)
After freeze:  xmin = 2 (FrozenTransactionId, always visible)
```

### Monitoring

```sql
-- Check how close tables are to wraparound
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC
LIMIT 10;

-- Danger threshold: age > 200 million
-- Emergency: age approaching 2 billion → autovacuum_freeze_max_age
```

If vacuum cannot keep up, PostgreSQL enters **emergency autovacuum** — and if that fails, it shuts down to prevent data corruption.

---

## MySQL InnoDB: Undo Log Approach

InnoDB takes a different MVCC approach: it updates tuples **in place** and stores old versions in an **undo log**.

```
Clustered index (B+tree):
┌───────────────────────────────────┐
│  Row: id=1, name='Bob'           │  ← Current version (in-place)
│  DB_TRX_ID=200, ROLL_PTR → undo │
└─────────────────────┬─────────────┘
                      │
                      ▼
Undo log:
┌───────────────────────────────────┐
│  Previous version: name='Alice'   │
│  TRX_ID=100, ROLL_PTR → older    │
└───────────────────────────────────┘
```

| Feature | PostgreSQL (heap) | MySQL InnoDB (undo log) |
|---------|-------------------|------------------------|
| Current version | May not be the first in chain | Always in the clustered index |
| Old versions | In the heap (same pages) | In undo tablespace (separate) |
| Update overhead | New tuple + dead tuple in heap | In-place update + undo entry |
| Cleanup | VACUUM on heap pages | Purge thread on undo log |
| Index updates | Required (new TID) | Not required if indexed columns unchanged |

---

## Key Takeaways

- MVCC enables **readers to never block writers** by maintaining multiple tuple versions — each transaction sees a consistent snapshot.
- PostgreSQL's append-only approach stores all versions in the heap, identified by `xmin`/`xmax` transaction IDs. InnoDB updates in place and stores old versions in an undo log.
- **Visibility rules** determine which tuple version a transaction can see based on its snapshot and the commit status of inserting/deleting transactions.
- **Hint bits** cache commit status in tuple headers to avoid repeated CLOG lookups.
- **Vacuum** is essential in PostgreSQL — it reclaims space from dead tuples, prevents index bloat, and avoids transaction ID wraparound.
- The **visibility map** accelerates both vacuum (skip all-visible pages) and index-only scans (avoid heap fetches).
- **Transaction ID wraparound** is a PostgreSQL-specific risk — vacuum must freeze old tuples before the 32-bit XID space wraps around.
