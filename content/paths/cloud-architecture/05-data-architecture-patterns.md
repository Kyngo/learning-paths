---
title: "Data Architecture Patterns"
weight: 5
---

# Data Architecture Patterns

Data outlives applications. The services you build today will be rewritten, but the data they produce will still matter in ten years. Cloud data architecture is about choosing the right storage for each workload, placing data where it needs to be, and ensuring it's protected, governed, and accessible.

---

## Polyglot Persistence

Using a single database for everything is an anti-pattern. Different data access patterns require different storage engines.

### Storage Selection Framework

| Access Pattern | Best Storage Type | Examples |
|---------------|------------------|----------|
| Transactional CRUD with joins | Relational (RDBMS) | PostgreSQL, MySQL, Cloud SQL |
| Key-value lookups, high throughput | Key-value store | DynamoDB, Bigtable, Cosmos DB |
| Full-text search, faceted queries | Search engine | OpenSearch, Elasticsearch |
| Hierarchical documents, flexible schema | Document store | MongoDB, Firestore, DocumentDB |
| Relationships and graph traversal | Graph database | Neptune, Neo4j, JanusGraph |
| Time-series metrics, IoT data | Time-series DB | TimescaleDB, InfluxDB, Timestream |
| Large file storage, media, backups | Object storage | S3, GCS, Azure Blob |
| In-memory caching, session state | Cache | Redis, Memcached, ElastiCache |
| Analytical queries, aggregations | Columnar / warehouse | Redshift, BigQuery, Snowflake |

### Polyglot Architecture Example

```
┌─────────────────────────────────────────────────────────────┐
│                     Booking Service                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ PostgreSQL   │  │ Redis        │  │ OpenSearch       │  │
│  │              │  │              │  │                  │  │
│  │ Bookings,    │  │ Session      │  │ Full-text search │  │
│  │ Users,       │  │ cache,       │  │ across bookings, │  │
│  │ Transactions │  │ rate limits  │  │ faceted filters  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │ S3           │  │ DynamoDB     │                         │
│  │              │  │              │                         │
│  │ Booking PDFs,│  │ Activity     │                         │
│  │ photos,      │  │ log, audit   │                         │
│  │ attachments  │  │ trail        │                         │
│  └──────────────┘  └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

### When NOT to Go Polyglot

| Situation | Recommendation |
|-----------|---------------|
| Small team (< 5 engineers) | Start with one database, add others only when bottlenecks prove it |
| Simple CRUD application | PostgreSQL handles 95% of use cases |
| No one on the team knows the new database | Operational complexity will eat your velocity |
| The performance problem is in the query, not the engine | Optimise indexes before adding a new database |

---

## Data Gravity

Data gravity is the principle that data attracts applications and services. Large datasets are expensive and slow to move, so applications tend to migrate toward data, not the other way around.

### Implications for Architecture

```
High Data Gravity                    Low Data Gravity
┌─────────────────────┐              ┌─────────────────────┐
│  10 TB data lake     │              │  100 MB config DB    │
│  in AWS us-east-1    │              │                     │
│                     │              │  Easy to replicate   │
│  Moving it = weeks  │              │  or move anywhere    │
│  + $900 egress      │              │                     │
│                     │              │                     │
│  → Applications     │              │  → Put it wherever   │
│    must come here   │              │    it's needed       │
└─────────────────────┘              └─────────────────────┘
```

**Design rules:**
1. Co-locate compute with high-gravity data — move the code, not the data
2. Factor egress costs into multi-region and multi-cloud decisions
3. Use data replication for read-heavy workloads instead of cross-region queries
4. Process data where it lives (edge computing, in-database analytics)

---

## Data Mesh

Data mesh is an organisational and architectural pattern that decentralises data ownership to domain teams rather than centralising it in a data engineering team.

### Four Principles of Data Mesh

| Principle | What It Means |
|-----------|--------------|
| **Domain ownership** | Each domain team owns, produces, and serves its data |
| **Data as a product** | Data is treated as a product with SLOs, documentation, and discoverability |
| **Self-serve platform** | A central platform team provides infrastructure for domains to publish and consume data |
| **Federated governance** | Global standards for interoperability; local autonomy for domain-specific decisions |

### Data Mesh Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    Data Mesh Platform                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │ Data       │  │ Schema     │  │ Access      │               │
│  │ Catalogue  │  │ Registry   │  │ Control     │               │
│  └────────────┘  └────────────┘  └────────────┘               │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Bookings Domain          Search Domain          Payments Domain│
│  ┌─────────────────┐     ┌────────────────┐    ┌─────────────┐│
│  │ Bookings DB     │     │ Search Index   │    │ Payments DB ││
│  │ (operational)   │     │ (operational)  │    │ (operational)││
│  │                 │     │                │    │             ││
│  │ Bookings Data   │     │ Search Data    │    │ Payments    ││
│  │ Product:        │     │ Product:       │    │ Data Product││
│  │ • bookings.v1   │     │ • queries.v1   │    │ • txns.v1   ││
│  │ • SLO: 99.9%    │     │ • SLO: 99.5%   │    │ • SLO: 99.9%││
│  └─────────────────┘     └────────────────┘    └─────────────┘│
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Data Mesh vs Centralised Data Team

| Aspect | Centralised | Data Mesh |
|--------|------------|-----------|
| Data ownership | Data engineering team | Domain teams |
| Bottleneck | Data team is the bottleneck | Domain teams are autonomous |
| Quality responsibility | Central team | Data producers |
| Best for | Small orgs, few domains | Large orgs, many autonomous teams |
| Risk | Single team can't understand all domains | Inconsistency without governance |

---

## Data Sovereignty and Residency

Data sovereignty dictates where data can be stored and processed based on regulatory requirements.

### Common Regulations

| Regulation | Scope | Key Requirement |
|-----------|-------|-----------------|
| GDPR (EU) | EU citizens' personal data | Data processing must have legal basis; data subject rights |
| Data Residency (various) | Country-specific | Data must be stored within national borders |
| HIPAA (US) | Health data | Encryption, access controls, BAA with provider |
| PCI DSS | Payment card data | Network segmentation, encryption, audit logging |
| CCPA (California) | California residents | Right to know, delete, opt-out of data sale |

### Architecture for Data Residency

```
┌─────────────────────────────────────────────────┐
│              Global Application                   │
│                                                   │
│  Request arrives → Extract user's region          │
│         │                                         │
│         ├── EU user → Route to eu-west-1         │
│         │            (data stays in EU)           │
│         │                                         │
│         ├── US user → Route to us-east-1         │
│         │            (data stays in US)           │
│         │                                         │
│         └── APAC user → Route to ap-southeast-1  │
│                        (data stays in APAC)       │
│                                                   │
│  Metadata and non-PII → Replicated globally       │
│  PII and regulated data → Region-locked           │
└─────────────────────────────────────────────────┘
```

---

## Storage Tiering

Not all data needs the same performance or durability. Storage tiering matches data to the right cost/performance tier based on access frequency.

### Tiering Strategy

| Tier | Access Frequency | Latency | Cost | Use Case |
|------|-----------------|---------|------|----------|
| **Hot** | Multiple times per day | Milliseconds | $$$ | Active databases, caches, session data |
| **Warm** | Weekly to monthly | Milliseconds | $$ | Recent logs, recent backups, reports |
| **Cold** | Quarterly or less | Seconds to minutes | $ | Old logs, compliance archives, older backups |
| **Archive** | Rarely (legal hold) | Hours | ¢ | 7-year retention, legal discovery, audit |

### Lifecycle Policies

```
Object created in S3 Standard (hot)
        │
        │ After 30 days, access drops
        ▼
Transition to S3 Infrequent Access (warm)
        │
        │ After 90 days, rarely accessed
        ▼
Transition to S3 Glacier Instant Retrieval (cold)
        │
        │ After 365 days, compliance only
        ▼
Transition to S3 Glacier Deep Archive (archive)
        │
        │ After 2555 days (7 years), regulatory period ends
        ▼
Delete (or retain if required)
```

---

## Cross-Region Replication Patterns

### Replication Strategies

| Strategy | Consistency | Latency | Cost | Use Case |
|----------|-----------|---------|------|----------|
| **Active-passive** | Eventual | Low reads in primary region | Low | DR, read replicas in secondary |
| **Active-active** | Eventual or conflict resolution | Low reads everywhere | High | Global users, low-latency everywhere |
| **Active-read** | Eventual | Low reads in all regions, writes to primary | Medium | Global reads, regional writes |

### Active-Active Architecture

```
┌──────────────┐         ┌──────────────┐
│  EU Region   │◄───────►│  US Region   │
│              │  Async   │              │
│  App + DB    │  Repli-  │  App + DB    │
│  (read/write)│  cation  │  (read/write)│
└──────┬───────┘         └──────┬───────┘
       │                        │
       │    ┌──────────────┐    │
       └───►│ APAC Region  │◄──┘
            │              │
            │  App + DB    │
            │  (read/write)│
            └──────────────┘

Conflict resolution: Last-writer-wins (LWW)
or application-level merge logic
```

### Conflict Resolution Approaches

| Approach | How It Works | Trade-Off |
|----------|-------------|-----------|
| Last-writer-wins (LWW) | Timestamp-based; latest write wins | Simple but loses data silently |
| Version vectors | Track causal history; detect conflicts | Complex, requires merge logic |
| Application-level merge | App-specific logic resolves conflicts | Most correct, most work |
| CRDTs | Mathematically guaranteed convergence | Limited data types, higher memory |

---

## Key Takeaways

- Use polyglot persistence: match the storage engine to the access pattern, but start simple and add complexity only when bottlenecks prove it
- Data gravity is real — co-locate compute with large datasets and factor egress costs into multi-region decisions
- Data mesh decentralises ownership to domain teams but requires a self-serve platform and federated governance
- Data sovereignty is a hard constraint — know your regulatory requirements and design region-locked storage from the start
- Storage tiering reduces costs by 60–90% for infrequently accessed data; automate transitions with lifecycle policies
- Cross-region replication introduces conflict resolution complexity — active-passive is simpler and sufficient for most DR requirements
