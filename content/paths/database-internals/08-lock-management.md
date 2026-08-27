---
title: "Lock Management"
weight: 8
---

## Why Locks Still Matter with MVCC

MVCC handles most read/write concurrency without locks, but locks are still essential for:

- **Write–write conflicts** — two transactions updating the same row need coordination
- **DDL operations** — schema changes must not conflict with concurrent queries
- **Explicit locking** — `SELECT ... FOR UPDATE`, advisory locks
- **Internal structures** — B+tree page latches, buffer pool frame pins

This section covers the lock manager: the subsystem that grants, queues, detects deadlocks, and escalates locks.

---

## Lock Modes

Databases define a hierarchy of lock modes with varying degrees of exclusivity.

### PostgreSQL Lock Modes (Table-Level)

| Mode | Conflicts with | Typical usage |
|------|---------------|---------------|
| `ACCESS SHARE` | `ACCESS EXCLUSIVE` only | `SELECT` |
| `ROW SHARE` | `EXCLUSIVE`, `ACCESS EXCLUSIVE` | `SELECT ... FOR UPDATE` |
| `ROW EXCLUSIVE` | `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE` |
| `SHARE UPDATE EXCLUSIVE` | `SHARE UPDATE EXCLUSIVE` and above | `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY` |
| `SHARE` | `ROW EXCLUSIVE` and above | `CREATE INDEX` (non-concurrent) |
| `SHARE ROW EXCLUSIVE` | `ROW EXCLUSIVE` and above | Rare (CREATE TRIGGER) |
| `EXCLUSIVE` | `ROW SHARE` and above | Rare |
| `ACCESS EXCLUSIVE` | Everything | `DROP TABLE`, `ALTER TABLE`, `VACUUM FULL` |

### Compatibility Matrix (Simplified)

```
                 Requested Lock
              SH    EX    UP
Held Lock  ┌─────┬─────┬─────┐
  SH       │  ✓  │  ✗  │  ✓  │   SH = Shared
  EX       │  ✗  │  ✗  │  ✗  │   EX = Exclusive
  UP       │  ✓  │  ✗  │  ✗  │   UP = Update
           └─────┴─────┴─────┘
  ✓ = compatible (both can be held simultaneously)
  ✗ = conflict (requester must wait)
```

### Update Lock

The **update lock** prevents a specific deadlock pattern:

```
Without update lock:
  T1: acquires SHARED lock on row (for read before update)
  T2: acquires SHARED lock on same row
  T1: tries to upgrade to EXCLUSIVE → blocked by T2's SHARED
  T2: tries to upgrade to EXCLUSIVE → blocked by T1's SHARED
  → DEADLOCK

With update lock:
  T1: acquires UPDATE lock (compatible with SHARED, not with UPDATE)
  T2: tries to acquire UPDATE lock → blocked (waits for T1)
  T1: upgrades to EXCLUSIVE → succeeds
  No deadlock.
```

---

## Lock Granularity

Locks can be acquired at different levels of the data hierarchy.

```
Lock Hierarchy:
┌──────────────────────┐
│    Database Lock      │
├──────────────────────┤
│    Table Lock         │
├──────────────────────┤
│    Page Lock          │
├──────────────────────┤
│    Row Lock           │
└──────────────────────┘
```

| Granularity | Concurrency | Overhead |
|-------------|-------------|----------|
| **Database** | Minimal | Very low |
| **Table** | Low | Low |
| **Page** | Medium | Medium |
| **Row** | High | High (one lock per row) |

### Trade-Off

- **Coarse locks** (table): few locks to manage, but block more concurrent transactions
- **Fine locks** (row): maximum concurrency, but managing millions of row locks consumes memory

### How PostgreSQL Handles Row Locks

PostgreSQL does not use the lock manager for row-level locks. Instead, it uses the tuple header (`xmax`) as an implicit lock:

```
T1: UPDATE users SET name='X' WHERE id=1
  → Sets xmax of the tuple to T1's transaction ID

T2: UPDATE users SET name='Y' WHERE id=1
  → Sees xmax is set to T1 (still active)
  → Waits for T1 to commit or abort
```

This means row locks do not consume lock table memory — they are stored directly in the heap pages.

---

## Lock Table Structure

The lock manager maintains a **lock table** — a hash table mapping lockable resources to their state.

```
Lock Table:
┌──────────────────────────────────────────────────┐
│  Resource: (table=users)                         │
│  Granted:  [T1: ROW EXCLUSIVE, T2: ROW EXCLUSIVE]│
│  Waiting:  [T3: EXCLUSIVE]                       │
├──────────────────────────────────────────────────┤
│  Resource: (table=orders)                        │
│  Granted:  [T4: ACCESS SHARE]                    │
│  Waiting:  []                                    │
└──────────────────────────────────────────────────┘
```

When a lock request arrives:
1. Check if the requested mode is compatible with all currently granted locks
2. If compatible: grant immediately
3. If not compatible: add to the wait queue

### FIFO Fairness

Most databases use a **FIFO wait queue** — requests are granted in the order they arrive. Without FIFO, exclusive lock requests could starve behind a continuous stream of shared locks.

---

## Intention Locks

When a transaction needs a row-level lock, how does the lock manager know that a table-level exclusive lock would conflict? Checking every row lock in the table is too expensive.

**Intention locks** solve this by signalling at higher levels what kind of lock is held at lower levels.

```
T1: UPDATE users SET name='X' WHERE id=1

Locks acquired:
  1. Intention Exclusive (IX) on table "users"  ← Signals: I hold/will hold exclusive at row level
  2. Exclusive (X) on row (users, id=1)

T2: ALTER TABLE users ADD COLUMN age INT

Lock requested:
  1. Access Exclusive on table "users"
  → Checks: any IX or IS on table? Yes (T1 has IX)
  → CONFLICT → T2 waits
```

### Intention Lock Compatibility

```
                    Requested
              IS    IX    S     X
Held    ┌─────┬─────┬─────┬─────┐
  IS    │  ✓  │  ✓  │  ✓  │  ✗  │
  IX    │  ✓  │  ✓  │  ✗  │  ✗  │
  S     │  ✓  │  ✗  │  ✓  │  ✗  │
  X     │  ✗  │  ✗  │  ✗  │  ✗  │
        └─────┴─────┴─────┴─────┘
  IS = Intention Shared   IX = Intention Exclusive
  S  = Shared             X  = Exclusive
```

---

## Deadlock Detection

A **deadlock** occurs when two or more transactions form a circular wait.

```
Deadlock:
  T1 holds lock on A, waits for lock on B
  T2 holds lock on B, waits for lock on A

Wait-for graph:
  T1 ──waits──→ T2
  T2 ──waits──→ T1    ← Cycle detected = deadlock
```

### Wait-for Graph

The lock manager maintains a directed graph:
- **Nodes:** transactions
- **Edges:** T1 → T2 means "T1 is waiting for a lock held by T2"

```
function detect_deadlock():
    build wait-for graph from lock table
    if graph contains a cycle:
        choose victim (usually the youngest/cheapest transaction)
        abort victim transaction
        log deadlock details
```

### Deadlock Detection Strategies

| Strategy | Method | Used by |
|----------|--------|---------|
| **Periodic detection** | Run cycle detection on a timer (e.g., every 1s) | PostgreSQL (100ms default), MySQL InnoDB |
| **Timeout** | Abort transaction after waiting too long | Simple but imprecise |
| **Prevention** | Order resources (e.g., always lock lower table OID first) | Application-level discipline |

### PostgreSQL Deadlock Detection

PostgreSQL checks for deadlocks after a transaction has waited for `deadlock_timeout` (default: 1 second). It performs a depth-first search on the wait-for graph. If a cycle is found, it aborts one transaction (choosing the one that would break the cycle with least work wasted).

```sql
-- View current locks and waits
SELECT pid, locktype, relation::regclass, mode, granted, waitstart
FROM pg_locks
WHERE NOT granted
ORDER BY waitstart;
```

---

## Lock Escalation

If a transaction acquires too many fine-grained locks (e.g., thousands of row locks), the lock manager may **escalate** to a coarser granularity to reduce memory usage.

```
Lock escalation:
  T1 holds 5,000 row locks on table "orders"
  Lock manager threshold: 5,000 locks
  → Escalate: release row locks, acquire TABLE lock instead

Before: 5,000 row locks (high concurrency, high memory)
After:  1 table lock (low concurrency, low memory)
```

| Database | Escalation behaviour |
|----------|---------------------|
| SQL Server | Row → page → table (automatic at ~5000 locks) |
| PostgreSQL | No escalation — row locks are in tuple headers (no lock table memory) |
| MySQL InnoDB | No explicit escalation — uses implicit row locks via undo records |
| Oracle | No automatic escalation — row-level only |

---

## Optimistic vs Pessimistic Concurrency

### Pessimistic (Lock-Based)

Assume conflicts are likely. Acquire locks before accessing data.

```
Pessimistic:
  T1: LOCK row → read → modify → COMMIT → release lock
  T2: LOCK row → BLOCKED until T1 commits
```

### Optimistic (Validation-Based)

Assume conflicts are rare. Read without locks, validate at commit time.

```
Optimistic:
  T1: read row (version=5) → modify locally
  T2: read row (version=5) → modify locally
  
  T1: COMMIT → check: row still version 5? Yes → write, set version=6
  T2: COMMIT → check: row still version 5? No (it's 6) → ABORT and retry
```

### Comparison

| Property | Pessimistic | Optimistic |
|----------|-------------|------------|
| Contention handling | Block on conflict | Retry on conflict |
| Best for | High contention (frequent conflicts) | Low contention (rare conflicts) |
| Overhead | Lock management, potential deadlocks | Validation, potential retries |
| Read-heavy workloads | Shared locks or MVCC snapshots | No overhead until write |
| Implementation | Lock manager + MVCC | Version numbers / timestamps |

### Optimistic in Practice

- **Application-level:** add a `version` column, check in `WHERE` clause on update
- **Database-level:** PostgreSQL's `SERIALIZABLE` isolation uses **Serializable Snapshot Isolation (SSI)**, which is optimistic — transactions execute without blocking but are aborted if a serialisation anomaly is detected at commit time

```sql
-- Application-level optimistic locking
UPDATE products
SET name = 'Widget', version = version + 1
WHERE id = 42 AND version = 5;
-- If 0 rows affected → conflict, retry
```

---

## Latches vs Locks

Databases distinguish between **locks** (transaction-level) and **latches** (internal, short-duration).

| Property | Lock | Latch |
|----------|------|-------|
| Purpose | Protect logical data (rows, tables) | Protect internal structures (pages, buffers) |
| Duration | Transaction lifetime | Microseconds to milliseconds |
| Deadlock detection | Yes | No (must be acquired in fixed order) |
| Visible to users | Yes (`pg_locks`, `SHOW ENGINE INNODB STATUS`) | No (internal implementation detail) |
| Example | Row lock during UPDATE | B+tree page latch during split |

---

## Key Takeaways

- Even with MVCC, locks are needed for **write–write conflicts**, **DDL operations**, and **explicit locking** (`SELECT ... FOR UPDATE`).
- **Lock modes** form a compatibility matrix — shared locks coexist, exclusive locks block everything, update locks prevent deadlocks during lock upgrades.
- **Intention locks** enable efficient coarse-to-fine-grained locking without checking every row lock in a table.
- **Deadlock detection** uses a wait-for graph; PostgreSQL runs cycle detection after `deadlock_timeout` (1s default) and aborts the cheapest transaction.
- **Lock escalation** trades concurrency for memory savings — SQL Server does it automatically; PostgreSQL avoids it by storing row locks in tuple headers.
- **Optimistic concurrency** (validate-on-commit) avoids lock overhead in low-contention scenarios; **pessimistic** (lock-on-access) is safer under high contention.
- **Latches** are internal, short-lived, and invisible to users — they protect B+tree nodes, buffer frames, and other internal structures.
