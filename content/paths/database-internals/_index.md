---
title: "Database Internals"
weight: 48
bookCollapseSection: true
---

# Database Internals

A deep dive into how databases work under the hood — storage engines, indexing structures, write-ahead logging, buffer management, concurrency control, and query processing. This path assumes you already know how to *use* a database and focuses entirely on the internal mechanisms that make it all work.

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Storage Fundamentals]({{< relref "01-storage-fundamentals" >}}) | Disk vs SSD, page-based storage, heap files, slotted pages, page layout, tuple structure, row vs column storage |
| 2 | [B-Trees & B+Trees]({{< relref "02-btrees" >}}) | Node structure, search/insert/delete, page splits and merges, fill factor, sibling pointers, bulk loading |
| 3 | [LSM Trees & Log-Structured Storage]({{< relref "03-lsm-trees" >}}) | Memtable, SSTable, write path, read path, compaction strategies, bloom filters, write vs read amplification |
| 4 | [Write-Ahead Logging]({{< relref "04-write-ahead-logging" >}}) | WAL purpose, log structure, LSN, ARIES crash recovery, checkpointing, physiological logging, group commit |
| 5 | [Buffer Pool Management]({{< relref "05-buffer-pool" >}}) | Page cache, buffer pool structure, eviction policies (LRU, clock), dirty pages, pin/unpin, prefetching |
| 6 | [MVCC Implementation]({{< relref "06-mvcc" >}}) | Tuple versioning, xmin/xmax, visibility rules, snapshot isolation, vacuum and garbage collection, visibility maps |
| 7 | [Query Processing]({{< relref "07-query-processing" >}}) | Parsing, planning, optimisation, cost estimation, cardinality estimation, join algorithms, plan trees, EXPLAIN |
| 8 | [Lock Management]({{< relref "08-lock-management" >}}) | Lock modes, lock granularity, deadlock detection, lock escalation, intention locks, optimistic vs pessimistic |
| 9 | [Index Internals]({{< relref "09-index-internals" >}}) | Hash indexes, GiST, GIN, BRIN, bitmap indexes, partial indexes, covering indexes, index-only scans, HOT updates |
| 10 | [Storage Engine Architecture]({{< relref "10-storage-engine-architecture" >}}) | InnoDB vs MyISAM, PostgreSQL heap, RocksDB, engine composition, embedded databases (SQLite, BoltDB, LevelDB) |

## Prerequisites

- Solid understanding of relational databases and SQL (the [Databases & SQL]({{< relref "/paths/databases" >}}) path)
- Familiarity with operating system concepts (memory, file I/O, processes)

## What You'll Be Able To Do

- Explain how data is physically stored on disk and in memory
- Trace the lifecycle of a write from client to durable storage
- Compare B-tree and LSM-tree trade-offs for different workloads
- Understand how MVCC provides concurrent reads without blocking writes
- Read query execution plans with knowledge of what the optimiser actually does
- Reason about lock contention, deadlocks, and concurrency strategies
- Evaluate storage engine choices for specific access patterns

## Recommended Reading

- *Database Internals* by Alex Petrov (O'Reilly)
- *Designing Data-Intensive Applications* by Martin Kleppmann (O'Reilly)
- PostgreSQL documentation — [Internals chapters](https://www.postgresql.org/docs/current/internals.html)
