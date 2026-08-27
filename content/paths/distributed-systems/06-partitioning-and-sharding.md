---
title: Partitioning & Sharding
weight: 6
---

# Partitioning & Sharding

When a dataset grows beyond what a single node can store or serve, it must be split across multiple nodes. **Partitioning** (also called **sharding**) divides data so each node is responsible for a subset. Combined with replication (Section 5), partitioning enables systems to scale both storage and throughput horizontally.

This section covers the strategies for partitioning data, the mechanisms for distributing partitions across nodes, and the problems that arise when you get it wrong.

## Why Partition?

| Goal | How partitioning helps |
|------|----------------------|
| **Storage** | Each node stores a fraction of the total data |
| **Throughput** | Queries are distributed across nodes |
| **Latency** | Data can be placed near the users who access it |
| **Isolation** | A hot partition affects only one shard, not the entire system |

The ideal partition scheme distributes data **evenly** and ensures that queries touch **as few partitions as possible**.

## Range Partitioning

Range partitioning assigns a contiguous range of keys to each partition.

```
Partition 1: keys A–F     → Node 1
Partition 2: keys G–M     → Node 2
Partition 3: keys N–S     → Node 3
Partition 4: keys T–Z     → Node 4
```

### Advantages

- **Range queries are efficient** — scanning keys `H` through `L` hits only Partition 2
- **Locality for related data** — keys with a common prefix are colocated
- **Predictable routing** — a sorted partition map makes lookup O(log N)

### Disadvantages

| Problem | Example |
|---------|---------|
| **Hot spots** | If keys are sequential (e.g., timestamps), all writes go to the last partition |
| **Uneven distribution** | Real data rarely has uniform key distribution |
| **Manual range management** | Ranges must be adjusted as data grows |

### Mitigating Hot Spots

- **Prefix salting:** prepend a hash prefix to the key (`hash(user_id) + timestamp` instead of `timestamp`)
- **Compound keys:** partition on one dimension, sort on another (Cassandra's partition key + clustering key)
- **Split hot ranges:** monitor partition sizes and split ranges that exceed a threshold (HBase, Spanner)

## Hash Partitioning

Hash partitioning applies a hash function to the key and assigns partitions based on hash ranges.

```
partition = hash(key) mod N

key="user:42"  → hash=0x7A3F → partition 0x7A3F mod 4 = 3 → Node 3
key="user:43"  → hash=0x1D02 → partition 0x1D02 mod 4 = 2 → Node 2
```

### Advantages

- **Even distribution** — a good hash function spreads keys uniformly regardless of key structure
- **No hot spots** from sequential keys

### Disadvantages

- **Range queries are impossible** — adjacent keys hash to different partitions
- **Rehashing problem** — adding a node changes `mod N`, causing massive data movement

### Range vs Hash: Comparison

| Property | Range | Hash |
|----------|-------|------|
| Range queries | ✅ Efficient | ❌ Scatter-gather |
| Even distribution | ❌ Key-dependent | ✅ By construction |
| Hot spots | ❌ Sequential keys | ✅ Rare (unless application-level skew) |
| Ordering | ✅ Preserved | ❌ Destroyed |
| Rebalancing | Manual range splits | Rehashing or consistent hashing |

## Consistent Hashing

Consistent hashing (Karger et al., 1997) solves the rehashing problem. When a node is added or removed, only **K/N** keys need to move (where K = total keys, N = number of nodes), instead of almost all keys.

### How It Works

```
Imagine a circular hash space (ring) from 0 to 2^32:

        ┌─── Node A (position 1000)
        │
        │         ┌─── Node B (position 4000)
        │         │
   ─────●─────────●──────────────── hash ring
   0    │         │                 2^32
        │         │
        │    ┌────●─── Node C (position 7000)
        │    │
        ●────┘
   Node D (position 9000)

Key placement: hash the key, walk clockwise, assign to first node.

key "x" → hash = 2500 → walk clockwise → Node B (4000)
key "y" → hash = 8000 → walk clockwise → Node D (9000)
```

### Adding a Node

```
Add Node E at position 5500:

Before: keys 4001–7000 → Node C
After:  keys 4001–5500 → Node E
        keys 5501–7000 → Node C

Only keys in the range 4001–5500 are moved. All other keys stay put.
```

### Virtual Nodes (Vnodes)

Physical nodes are mapped to **multiple positions** on the ring. This improves balance and makes rebalancing smoother.

```
Node A → positions {1000, 3500, 6200}   (3 vnodes)
Node B → positions {2000, 5000, 8000}   (3 vnodes)
Node C → positions {500, 4200, 7500}    (3 vnodes)

More vnodes = more even distribution
Typical: 256 vnodes per physical node
```

### Consistent Hashing in Practice

| System | Consistent hashing variant |
|--------|--------------------------|
| Amazon DynamoDB | Modified consistent hashing with virtual nodes |
| Apache Cassandra | Token ring with vnodes |
| Riak | Ring-based with virtual nodes |
| Memcached (client-side) | Ketama consistent hashing |

## Rebalancing Strategies

As the cluster grows or shrinks, partitions must be redistributed. This is called **rebalancing**.

### Fixed Number of Partitions

Create many more partitions than nodes from the start. Assign multiple partitions to each node. When a node is added, move some partitions from existing nodes.

```
Initial: 16 partitions, 4 nodes → 4 partitions each
Add node: 16 partitions, 5 nodes → ~3 partitions each
         (move 3-4 partitions to the new node)
```

| System | Fixed partitions? | Default count |
|--------|------------------|---------------|
| Riak | Yes | 64 |
| Elasticsearch | Yes (at index creation) | 1 per index (configurable) |
| Couchbase | Yes | 1024 |

**Tradeoff:** the number of partitions must be chosen upfront. Too few = large partitions, slow rebalancing. Too many = high metadata overhead.

### Dynamic Partitioning

When a partition grows beyond a threshold, split it. When it shrinks, merge it. The number of partitions adapts to the data size.

```
Partition [A–M] grows to 10 GB (threshold: 8 GB)
    → Split into [A–F] and [G–M]

Partition [A–F] shrinks to 500 MB (min threshold: 1 GB)
    → Merge with neighbour
```

Used by **HBase**, **MongoDB**, and **Spanner**. Works well with range partitioning where splits are natural.

### Proportional to Node Count

Maintain a fixed number of partitions **per node**. When a node joins, it splits a random selection of existing partitions and takes half.

Used by **Cassandra** (with vnodes). Keeps partition count proportional to cluster capacity.

## Secondary Indexes and Partitioning

Partitioning complicates secondary indexes because the indexed field may not align with the partition key.

### Local Indexes (Document-Partitioned)

Each partition maintains its own index covering only the data **in that partition**.

```
Partition 1 (users A–M):
    Primary: {alice, bob, charlie}
    Index on city: {london: [alice, charlie], paris: [bob]}

Partition 2 (users N–Z):
    Primary: {nora, sam, tina}
    Index on city: {london: [sam], berlin: [nora, tina]}

Query "WHERE city = london":
    → Must query BOTH partitions (scatter-gather)
    → Merge results
```

**Pro:** writes are local (fast). **Con:** reads must scatter to all partitions (slow for secondary index queries).

### Global Indexes (Term-Partitioned)

The secondary index itself is partitioned, but differently from the primary data.

```
Index partition 1 (cities A–L):
    berlin: [nora@P2, tina@P2]
    london: [alice@P1, charlie@P1, sam@P2]

Index partition 2 (cities M–Z):
    paris: [bob@P1]

Query "WHERE city = london":
    → Query only index partition 1
    → Locate: alice@P1, charlie@P1, sam@P2
    → Fetch from primary partitions P1 and P2
```

**Pro:** reads touch fewer partitions. **Con:** writes must update the global index across partitions (slower, requires distributed coordination).

### Comparison

| Property | Local index | Global index |
|----------|------------|-------------|
| Write cost | Low (local) | High (cross-partition) |
| Read cost (secondary query) | High (scatter-gather all partitions) | Low (query relevant index partitions) |
| Consistency | Immediate (same partition) | Async (eventually consistent) or expensive (synchronous) |
| Used by | MongoDB, Cassandra, Elasticsearch | DynamoDB (GSI), Google Spanner |

## Hot Spots

Even with good partitioning, application-level access patterns can create hot spots.

### Causes

| Cause | Example |
|-------|---------|
| Celebrity problem | A viral tweet causes millions of reads on one partition |
| Temporal locality | Today's data is read 1000x more than yesterday's |
| Natural skew | 1% of customers generate 50% of transactions |

### Mitigations

| Technique | How it works | Tradeoff |
|-----------|-------------|----------|
| **Key salting** | Append random suffix (e.g., `key_0`, `key_1` ... `key_9`) | Spreads writes; reads must scatter to 10 keys and merge |
| **Read replicas for hot keys** | Detect hot keys, add read-only replicas | Complexity; requires hot key detection |
| **Application-level caching** | Cache hot data at the application tier | Stale reads; cache invalidation complexity |
| **Split hot partitions** | Dynamically split partitions with high load | Requires monitoring and automation |

### Hot Key Detection

```
Approach: Monitor per-partition metrics (ops/sec, bytes/sec)

If partition P exceeds threshold:
    1. Identify the hot key(s) within P
    2. Apply mitigation:
       - Salt the key to spread across partitions
       - Add read replicas for the key
       - Cache at application level
    3. Alert operators for review
```

## Key Takeaways

- Partitioning splits data across nodes for scalability — the goal is even distribution with minimal cross-partition queries
- Range partitioning preserves key ordering (good for scans) but risks hot spots on sequential keys
- Hash partitioning distributes evenly but destroys key ordering — range queries become scatter-gather operations
- Consistent hashing minimises data movement when nodes are added or removed — virtual nodes improve balance
- Rebalancing strategies include fixed partition count (simple), dynamic splitting (adaptive), and proportional-to-nodes (Cassandra)
- Secondary indexes on partitioned data require a choice between local indexes (fast writes, slow reads) and global indexes (slow writes, fast reads)
- Hot spots occur despite good partitioning — detection and mitigation (salting, caching, splitting) must be built into operations
