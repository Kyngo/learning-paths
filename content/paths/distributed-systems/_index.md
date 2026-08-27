---
title: Distributed Systems
weight: 51
bookCollapseSection: true
---

# Distributed Systems

A deep dive into the theory and practice of distributed systems — the algorithms, consistency models, and failure modes that underpin every large-scale system. This path focuses on the foundational concepts that are often hand-waved in higher-level system design discussions.

## Prerequisites

This path assumes familiarity with:

- **System Design** fundamentals (load balancing, caching, microservices) — covered in the [System Design]({{< relref "/paths/system-design" >}}) path
- **CAP theorem** — covered in [Databases: NoSQL]({{< relref "/paths/databases/09-nosql" >}})
- **Event sourcing & CQRS** — covered in [Software Architecture: Event-Driven Architecture]({{< relref "/paths/software-architecture/05-event-driven-architecture" >}})
- **Circuit breaker, retry & bulkhead patterns** — covered in [System Design: Reliability & Resilience]({{< relref "/paths/system-design/09-reliability-resilience" >}})

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Distributed Systems Fundamentals]({{< relref "01-fundamentals" >}}) | Why distribution is hard, fallacies of distributed computing, network partitions, partial failures, FLP impossibility, system models |
| 2 | [Time & Ordering]({{< relref "02-time-and-ordering" >}}) | Physical clocks, clock skew, Lamport clocks, vector clocks, hybrid logical clocks, happens-before, causal ordering |
| 3 | [Consistency Models]({{< relref "03-consistency-models" >}}) | Linearizability, sequential consistency, causal consistency, eventual consistency, strong eventual consistency, session guarantees |
| 4 | [Consensus Algorithms]({{< relref "04-consensus-algorithms" >}}) | Paxos, Multi-Paxos, Raft, Viewstamped Replication, ZAB, practical comparison |
| 5 | [Replication]({{< relref "05-replication" >}}) | Single-leader, multi-leader, leaderless quorum, read-your-writes, monotonic reads, chain replication, conflict resolution |
| 6 | [Partitioning & Sharding]({{< relref "06-partitioning-and-sharding" >}}) | Range partitioning, hash partitioning, consistent hashing, rebalancing, secondary indexes, hot spots |
| 7 | [Distributed Transactions]({{< relref "07-distributed-transactions" >}}) | Two-phase commit, three-phase commit, distributed locking, Calvin, Spanner TrueTime, transaction coordinators |
| 8 | [CRDTs & Eventual Consistency]({{< relref "08-crdts" >}}) | State-based vs operation-based CRDTs, G-Counter, PN-Counter, OR-Set, LWW-Register, Merkle trees, anti-entropy |
| 9 | [Failure Detection & Recovery]({{< relref "09-failure-detection-and-recovery" >}}) | Heartbeats, phi accrual detector, gossip protocols, SWIM, membership protocols, read repair, hinted handoff |
| 10 | [Case Studies]({{< relref "10-case-studies" >}}) | Dynamo, Spanner, Kafka, CockroachDB, Cassandra, etcd — architecture decisions and tradeoffs |
