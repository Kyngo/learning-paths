---
title: "Storage Engine Architecture"
weight: 10
---

## What Is a Storage Engine?

A storage engine is the component responsible for **how data is physically stored, retrieved, and modified**. It sits below the SQL layer and above the operating system, composing several subsystems covered in earlier sections.

```
┌─────────────────────────────────────────────┐
│               SQL Layer                      │
│  (Parser → Planner → Optimiser → Executor)  │
├─────────────────────────────────────────────┤
│             Storage Engine                   │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │ Access   │ │ Buffer   │ │ Transaction │ │
│  │ Methods  │ │ Pool     │ │ Manager     │ │
│  │ (B+tree, │ │          │ │ (WAL, MVCC, │ │
│  │  heap,   │ │          │ │  locks)     │ │
│  │  LSM)    │ │          │ │             │ │
│  └──────────┘ └──────────┘ └─────────────┘ │
├─────────────────────────────────────────────┤
│         File System / Direct I/O             │
├─────────────────────────────────────────────┤
│              Storage Device                  │
└─────────────────────────────────────────────┘
```

---

## Engine Composition

Every storage engine combines the same fundamental subsystems — the differences lie in the choices each subsystem makes.

### Subsystem Map

| Subsystem | Responsibility | Choices |
|-----------|---------------|---------|
| **Access method** | On-disk data structure for tables and indexes | B+tree, LSM-tree, heap + indexes |
| **Buffer pool** | Page caching in memory | LRU, clock sweep, direct I/O |
| **WAL** | Crash recovery and durability | Redo-only, undo/redo (ARIES), WAL-per-table |
| **Concurrency control** | Transaction isolation | MVCC (append), MVCC (undo log), 2PL |
| **Lock manager** | Write–write coordination | Row locks, page locks, implicit (xmax) |
| **Compaction / vacuum** | Reclaiming dead space | Background vacuum, compaction, in-place |

---

## MySQL: InnoDB vs MyISAM

MySQL's **pluggable storage engine** architecture lets different tables use different engines. The SQL layer is shared; the engine handles storage.

```
MySQL Architecture:
┌──────────────────────────────────┐
│        MySQL Server Layer         │
│  (Parser, Optimiser, Executor)   │
├────────────┬─────────────────────┤
│  InnoDB    │  MyISAM             │
│  Engine    │  Engine             │
│            │                     │
│  B+tree    │  B+tree indexes    │
│  clustered │  + heap data files  │
│  MVCC      │  Table locks only   │
│  WAL       │  No crash recovery  │
│  Row locks │  No transactions    │
└────────────┴─────────────────────┘
```

### InnoDB

| Feature | InnoDB |
|---------|--------|
| Primary structure | **Clustered B+tree** (data stored in leaf nodes of primary key index) |
| Secondary indexes | Point to primary key value (not TID) — requires PK lookup |
| Concurrency | MVCC with undo log + row-level locks |
| Durability | Redo log (WAL) + doublewrite buffer |
| Buffer pool | Own buffer pool (`innodb_buffer_pool_size`), uses O_DIRECT |
| Crash recovery | Redo log replay + undo rollback |
| Foreign keys | Supported |

### InnoDB Clustered Index

```
Primary key B+tree (clustered):
         ┌─────────────┐
         │  [5 | 20]   │  ← Internal nodes: PK values
         └──┬──────┬───┘
     ┌──────┘      └──────┐
     ▼                    ▼
┌───────────────┐  ┌───────────────┐
│ PK=1: row data│  │ PK=5: row data│  ← Leaf nodes contain the full row
│ PK=2: row data│  │ PK=10: row... │
│ PK=3: row data│  │ PK=15: row... │
└───────────────┘  └───────────────┘

Secondary index:
         ┌───────────────┐
         │  [email vals] │
         └──────┬────────┘
                ▼
  ┌──────────────────────────┐
  │ email='alice' → PK=3    │  ← Points to PK, not page/slot
  │ email='bob'   → PK=1    │
  └──────────────────────────┘

Secondary index lookup = index scan + PK lookup (double traversal)
```

### MyISAM

| Feature | MyISAM |
|---------|--------|
| Primary structure | Heap file (`.MYD`) + separate B+tree indexes (`.MYI`) |
| Concurrency | **Table-level locks only** (no row locking) |
| Durability | No WAL, no crash recovery (relies on `REPAIR TABLE`) |
| Transactions | Not supported |
| Full-text search | Yes (legacy; InnoDB now supports it too) |
| Use case | Read-heavy, non-transactional (largely deprecated) |

### Why InnoDB Won

MyISAM's table-level locking made it unusable for concurrent write workloads. InnoDB became MySQL's default engine in version 5.5 (2010) and MyISAM is now legacy.

---

## PostgreSQL: Heap Storage

PostgreSQL does not have pluggable storage engines in the MySQL sense (though the **Table Access Method API** introduced in v12 enables alternatives). The default is a **heap-based** storage engine.

```
PostgreSQL Storage:
┌───────────────────────────────┐
│  Heap table (unordered pages) │
│  ┌──────┬──────┬──────┐      │
│  │Page 0│Page 1│Page 2│...   │
│  └──────┴──────┴──────┘      │
├───────────────────────────────┤
│  B+tree indexes (separate)   │
│  Index entries → TID (page, slot) │
├───────────────────────────────┤
│  WAL (pg_wal/)               │
├───────────────────────────────┤
│  MVCC via tuple headers      │
│  (xmin, xmax in heap tuples) │
└───────────────────────────────┘
```

### PostgreSQL vs InnoDB Architecture

| Feature | PostgreSQL (heap) | MySQL InnoDB (clustered) |
|---------|-------------------|-------------------------|
| Table structure | Unordered heap | Clustered B+tree (ordered by PK) |
| Primary key lookup | Index scan → heap fetch | Direct (data is in the PK index) |
| Secondary index target | TID (page, slot) | Primary key value |
| Secondary index overhead | One heap fetch | One PK B+tree traversal |
| Update (non-indexed col) | HOT if same page | In-place (no index update) |
| Dead tuple handling | VACUUM | Purge thread (undo log) |
| Table ordering | Not guaranteed | Ordered by PK |

### Table Access Method API

PostgreSQL v12 introduced an extensible API allowing alternative storage:

- **zheap** — undo-log-based MVCC (avoids bloat from dead tuples)
- **Citus Columnar** — columnar storage for analytical workloads
- **orioledb** — undo-log MVCC with index-organised tables

```sql
-- Use an alternative table access method
CREATE TABLE analytics_data (...) USING columnar;
```

---

## RocksDB

RocksDB (developed by Facebook, forked from LevelDB) is an **LSM-tree based embedded key-value store** used as a storage engine in many distributed databases.

```
RocksDB Architecture:
┌──────────────────────────────────────────┐
│  Application (CockroachDB, TiKV, etc.)  │
├──────────────────────────────────────────┤
│  RocksDB API                             │
│  Put(key, value) / Get(key) / Delete(key)│
├──────────────────────────────────────────┤
│  Memtable (skip list or hash-skip list)  │
├──────────────────────────────────────────┤
│  WAL (write-ahead log per column family) │
├──────────────────────────────────────────┤
│  SST files (sorted string tables)        │
│  Level 0: overlapping                    │
│  Level 1+: non-overlapping (leveled)     │
├──────────────────────────────────────────┤
│  Bloom filters (per-SST or per-block)    │
│  Block cache (LRU)                       │
│  Compression (Snappy, LZ4, Zstd)        │
└──────────────────────────────────────────┘
```

### RocksDB as a Building Block

| System | Uses RocksDB for |
|--------|-----------------|
| CockroachDB | Distributed SQL storage layer (migrating to Pebble, a Go rewrite) |
| TiKV (TiDB) | Distributed key-value storage |
| YugabyteDB | DocDB storage engine |
| Kafka (KRaft) | Metadata storage |
| MyRocks | MySQL storage engine alternative to InnoDB |

---

## Embedded Databases

Embedded databases run **inside the application process** — no separate server, no network protocol.

### SQLite

The world's most deployed database (billions of installations in phones, browsers, apps).

```
SQLite Architecture:
┌──────────────────────────────┐
│  Application Process         │
│  ┌──────────────────────┐   │
│  │  SQLite Library       │   │
│  │  ┌──────────────────┐│   │
│  │  │ B-tree (tables   ││   │
│  │  │ and indexes)     ││   │
│  │  ├──────────────────┤│   │
│  │  │ Pager (page      ││   │
│  │  │ cache + WAL)     ││   │
│  │  ├──────────────────┤│   │
│  │  │ OS Interface     ││   │
│  │  └──────────────────┘│   │
│  └──────────────────────┘   │
│              │               │
│         Single File          │
│        (database.db)         │
└──────────────────────────────┘
```

| Feature | SQLite |
|---------|--------|
| Storage | Single file (B-tree pages) |
| Concurrency | Multiple readers, single writer (file-level lock) |
| WAL mode | Multiple readers + single writer with WAL file |
| Page size | 4 KB default (configurable) |
| Transaction | Full ACID (serialised writes) |
| Use case | Mobile apps, desktop apps, edge computing, testing |

### BoltDB

A pure-Go embedded key-value store (used by etcd, Consul).

| Feature | BoltDB |
|---------|--------|
| Storage | Single file, B+tree |
| Concurrency | Multiple readers, single writer (MVCC via copy-on-write) |
| Transaction | Full ACID |
| Use case | Embedded metadata storage for Go applications |

### LevelDB

Google's embedded key-value store (inspiration for RocksDB).

| Feature | LevelDB |
|---------|---------|
| Storage | LSM-tree (memtable + sorted files) |
| Concurrency | Single process (no multi-process access) |
| Compaction | Leveled |
| Use case | Chrome's IndexedDB backend, embedded applications |

---

## Comparison: Engine Families

| Engine Type | Data Structure | Best For | Weakness |
|-------------|---------------|----------|----------|
| **Heap + B+tree** (PostgreSQL) | Unordered heap + separate indexes | General purpose, flexible | Tuple bloat (MVCC in heap) |
| **Clustered B+tree** (InnoDB) | Data in PK index leaves | PK lookups, range scans on PK | Secondary index double-lookup |
| **LSM-tree** (RocksDB, LevelDB) | Memtable + SSTables | Write-heavy, time-series | Read amplification, compaction overhead |
| **B-tree single-file** (SQLite, BoltDB) | B-tree pages in one file | Embedded, edge, testing | Limited concurrency |

---

## How Engines Compose: Summary Diagram

```
┌────────────────────────────────────────────────┐
│              Query Layer (SQL)                  │
│  Parse → Analyse → Rewrite → Plan → Execute    │
├─────────────┬──────────────┬───────────────────┤
│ Access      │ Buffer Pool  │ Transaction       │
│ Methods     │              │ Manager           │
│             │              │                   │
│ ┌─────────┐ │ ┌──────────┐ │ ┌───────────────┐ │
│ │ Heap    │ │ │ Page     │ │ │ WAL           │ │
│ │ B+tree  │ │ │ cache    │ │ │ (redo/undo)   │ │
│ │ GiST    │ │ │ Eviction │ │ ├───────────────┤ │
│ │ GIN     │ │ │ Dirty    │ │ │ MVCC          │ │
│ │ BRIN    │ │ │ flush    │ │ │ (visibility)  │ │
│ │ Hash    │ │ │ Prefetch │ │ ├───────────────┤ │
│ └─────────┘ │ └──────────┘ │ │ Lock Manager  │ │
│             │              │ │ (deadlock det.)│ │
│             │              │ └───────────────┘ │
├─────────────┴──────────────┴───────────────────┤
│           File System / Direct I/O              │
├────────────────────────────────────────────────┤
│              Storage (SSD/HDD)                  │
└────────────────────────────────────────────────┘
```

Each subsystem depends on the others:
- The **access method** requests pages from the **buffer pool**
- The **buffer pool** respects the **WAL** protocol before flushing dirty pages
- The **transaction manager** uses the **lock manager** for write coordination and the **WAL** for durability
- The **access method** checks **MVCC** visibility before returning tuples

---

## Choosing a Storage Engine

| Workload | Recommended Engine | Why |
|----------|-------------------|-----|
| General OLTP | PostgreSQL (heap) or InnoDB | Mature MVCC, row-level locking, full SQL |
| PK-heavy lookups | InnoDB (clustered) | Data is in the PK index — zero heap fetch |
| Write-heavy (logs, events) | LSM-based (RocksDB, Cassandra) | Sequential writes, high throughput |
| Embedded / edge | SQLite | Zero-config, single file, ACID |
| Analytics (columnar) | ClickHouse, DuckDB, Citus Columnar | Column store, compression, vectorised execution |
| Distributed SQL | CockroachDB (Pebble), TiDB (TiKV) | LSM + Raft consensus |

---

## Key Takeaways

- A storage engine composes **access methods** (B+tree, LSM, heap), a **buffer pool**, a **WAL**, **MVCC**, and a **lock manager** — each making trade-offs that define the engine's personality.
- MySQL's **pluggable architecture** lets different tables use different engines; InnoDB's clustered B+tree with MVCC and row locking made it the universal default over MyISAM.
- PostgreSQL uses a **heap + separate B+tree indexes** with MVCC in tuple headers. The Table Access Method API (v12+) opens the door to alternative engines.
- **RocksDB** is the dominant LSM-tree engine for distributed databases — it provides a key-value API that CockroachDB, TiDB, and others build SQL layers on top of.
- **Embedded databases** (SQLite, BoltDB, LevelDB) eliminate the server process, running entirely within the application — suitable for edge computing, mobile, and metadata storage.
- No single engine is best for all workloads. Understanding how the subsystems compose lets you match the engine to the access pattern.
