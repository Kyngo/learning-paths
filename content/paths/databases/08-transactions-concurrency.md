---
title: "Transactions and Concurrency"
weight: 8
---

## What Is a Transaction?

A **transaction** is a group of SQL statements that execute as a single unit — either all succeed or all fail. Transactions protect data integrity when multiple operations must happen together.

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;   -- Debit sender
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;   -- Credit receiver
COMMIT;
-- Both happen, or neither happens
```

If anything fails between BEGIN and COMMIT:
```sql
ROLLBACK;  -- Undo everything since BEGIN
```

---

## ACID Properties

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All or nothing — partial failures are rolled back | Transfer: both debit and credit succeed, or neither |
| **Consistency** | Database moves from one valid state to another | Constraints and rules are never violated |
| **Isolation** | Concurrent transactions don't interfere | Two transfers from same account don't overdraw |
| **Durability** | Once committed, data survives crashes | After COMMIT, data is on disk permanently |

---

## Transaction Syntax

```sql
-- Explicit transaction
BEGIN;  -- or: START TRANSACTION
    INSERT INTO orders (customer_id, total) VALUES (1, 99.99);
    INSERT INTO order_items (order_id, product_id, qty) VALUES (currval('orders_id_seq'), 5, 2);
    UPDATE products SET stock = stock - 2 WHERE id = 5;
COMMIT;

-- With error handling (PostgreSQL)
BEGIN;
    -- operations...
SAVEPOINT before_risky_operation;
    -- risky operation...
    -- If it fails:
ROLLBACK TO SAVEPOINT before_risky_operation;
    -- continue with alternative...
COMMIT;
```

### Auto-Commit

By default, each SQL statement is its own transaction (auto-commit). Explicit BEGIN/COMMIT groups multiple statements.

---

## Concurrency Problems

When multiple transactions run simultaneously, these problems can occur:

### Dirty Read

Transaction A reads uncommitted data from Transaction B. If B rolls back, A has read data that never existed.

```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;   (not committed)
T2: SELECT balance FROM accounts WHERE id = 1;       → reads 0 (dirty!)
T1: ROLLBACK;                                         → balance is actually 1000
```

### Non-Repeatable Read

Transaction A reads a row twice and gets different values because Transaction B modified it in between.

```
T1: SELECT balance FROM accounts WHERE id = 1;  → 1000
T2: UPDATE accounts SET balance = 500 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  → 500 (different!)
```

### Phantom Read

Transaction A runs a query twice and gets different rows because Transaction B inserted/deleted rows.

```
T1: SELECT COUNT(*) FROM employees WHERE dept = 'Eng';  → 10
T2: INSERT INTO employees (dept) VALUES ('Eng'); COMMIT;
T1: SELECT COUNT(*) FROM employees WHERE dept = 'Eng';  → 11 (phantom!)
```

### Lost Update

Two transactions read the same value, compute a new value, and write — the second write overwrites the first.

```
T1: SELECT stock FROM products WHERE id = 5;  → 10
T2: SELECT stock FROM products WHERE id = 5;  → 10
T1: UPDATE products SET stock = 10 - 1 = 9;   COMMIT;
T2: UPDATE products SET stock = 10 - 1 = 9;   COMMIT;  → Should be 8!
```

---

## Isolation Levels

Isolation levels control which concurrency problems are allowed:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|-----------|--------------------:|-------------:|-------------|
| **Read Uncommitted** | Possible | Possible | Possible | Fastest |
| **Read Committed** | Prevented | Possible | Possible | Good (default in PostgreSQL) |
| **Repeatable Read** | Prevented | Prevented | Possible* | Moderate |
| **Serializable** | Prevented | Prevented | Prevented | Slowest |

*PostgreSQL's Repeatable Read also prevents phantom reads (it uses snapshot isolation).

```sql
-- Set for current transaction
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
    -- operations...
COMMIT;

-- Set for session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### Choosing an Isolation Level

| Use Case | Recommended Level |
|----------|------------------|
| Most web applications | Read Committed (default) |
| Financial calculations | Repeatable Read or Serializable |
| Reports (consistent snapshot) | Repeatable Read |
| Inventory management | Serializable (or explicit locking) |

---

## Locking

### Implicit Locks

Databases automatically acquire locks to enforce isolation:

| Lock Type | Acquired By | Blocks |
|-----------|------------|--------|
| **Row-level shared (FOR SHARE)** | SELECT ... FOR SHARE | Other writers (not readers) |
| **Row-level exclusive (FOR UPDATE)** | UPDATE, DELETE, SELECT ... FOR UPDATE | All other access to that row |
| **Table-level** | DDL (ALTER, DROP) | All operations on the table |

### Explicit Locking

```sql
-- Lock rows for update (prevent concurrent modification)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- No other transaction can modify this row until we commit
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Skip locked rows (useful for job queues)
SELECT * FROM tasks WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- Advisory locks (application-level)
SELECT pg_advisory_lock(12345);    -- Acquire
-- ... do work ...
SELECT pg_advisory_unlock(12345);  -- Release
```

---

## Deadlocks

A deadlock occurs when two transactions wait for each other:

```
T1: UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Locks row 1
T2: UPDATE accounts SET balance = balance - 50  WHERE id = 2;  -- Locks row 2
T1: UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Waits for T2...
T2: UPDATE accounts SET balance = balance + 50  WHERE id = 1;  -- Waits for T1... DEADLOCK!
```

The database detects deadlocks and kills one transaction (the "victim") with an error. The application must retry.

### Preventing Deadlocks

1. **Access tables/rows in consistent order** — if all transactions lock row 1 before row 2, deadlocks can't occur
2. **Keep transactions short** — reduce the window for conflicts
3. **Use appropriate isolation level** — don't over-isolate
4. **Retry on deadlock error** — application must handle this case

---

## MVCC (Multi-Version Concurrency Control)

PostgreSQL and many modern databases use **MVCC** instead of read locks. Every transaction sees a consistent **snapshot** of the data:

- Readers never block writers
- Writers never block readers
- Each transaction sees the database as it was at transaction start (for Repeatable Read)
- Multiple versions of a row coexist

This is why PostgreSQL needs `VACUUM` — old row versions must be cleaned up after all transactions that could see them have finished.

```sql
-- Manual vacuum (usually autovacuum handles this)
VACUUM ANALYZE employees;

-- Check for bloat
SELECT relname, n_dead_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000;
```

---

## Key Takeaways

1. **Use transactions** for any operation that involves multiple related writes — bank transfers, order creation, inventory updates
2. **Read Committed** is the right default for most applications — don't over-isolate without reason
3. **SELECT ... FOR UPDATE** prevents lost updates — use it when you read-then-write based on the read value
4. **Keep transactions short** — long transactions hold locks, block others, and increase deadlock risk
5. **Always handle deadlock errors** — they're normal in concurrent systems, not bugs. Retry the transaction.
6. **MVCC** means readers and writers don't block each other in PostgreSQL — a major performance advantage
