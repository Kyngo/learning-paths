---
title: "Cloud Migration Strategies"
weight: 6
---

# Cloud Migration Strategies

Migration is not a technical project — it's a business transformation with technical components. Most migration failures are caused by poor planning, not poor execution. This section covers the strategic patterns for moving workloads to the cloud and the tactical patterns for cutting over with minimal risk.

---

## The 6 Rs of Migration

Every workload in your portfolio maps to one of six strategies. The right strategy depends on the workload's business value, technical complexity, and cloud readiness.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        The 6 Rs Spectrum                             │
│                                                                      │
│  Low Effort / Low Benefit ◄──────────────────► High Effort / Benefit │
│                                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ Retain   │ │ Retire   │ │ Rehost   │ │Replatform│ │ Refactor │ │
│  │          │ │          │ │ (Lift &  │ │ (Lift &  │ │ (Re-     │ │
│  │ Keep     │ │ Shut     │ │  Shift)  │ │  Tinker) │ │  write)  │ │
│  │ on-prem  │ │ down     │ │          │ │          │ │          │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│                                                                      │
│                             ┌──────────┐                             │
│                             │Repurchase│                             │
│                             │ (Replace │                             │
│                             │  w/ SaaS)│                             │
│                             └──────────┘                             │
└─────────────────────────────────────────────────────────────────────┘
```

### Strategy Comparison

| Strategy | What | When | Risk | Cloud Benefit |
|----------|------|------|------|--------------|
| **Retain** | Keep on-prem, no change | Regulatory, mainframe, too costly to move | None (no change) | None |
| **Retire** | Decommission | Redundant, unused, replaced | Low | Cost savings (remove hosting) |
| **Rehost** | Move VMs as-is to cloud VMs | Quick migration, low risk appetite | Low | Elasticity, no hardware refresh |
| **Replatform** | Minor changes to use managed services | Database → RDS, app server → containers | Medium | Reduced ops, managed patching |
| **Refactor** | Rewrite to cloud-native | Strategic apps needing scale, agility | High | Full cloud benefits |
| **Repurchase** | Replace with SaaS | Email → O365, CRM → Salesforce, ITSM → ServiceNow | Medium | Zero ops, vendor manages all |

### Decision Framework

```
Is the workload still needed?
├── No → RETIRE
└── Yes
    ├── Is there a SaaS replacement that fits?
    │   └── Yes → REPURCHASE
    └── No
        ├── Can it run unmodified in the cloud?
        │   ├── Yes → REHOST (quick win)
        │   └── Mostly → REPLATFORM (swap DB, containerise)
        └── Is it strategically important?
            ├── Yes → REFACTOR (invest in cloud-native)
            └── No → RETAIN (revisit later)
```

---

## Migration Wave Planning

You don't migrate everything at once. Migrations happen in waves, ordered by dependency, complexity, and business priority.

### Wave Prioritisation Matrix

| Factor | Weight | How to Assess |
|--------|--------|---------------|
| Business criticality | High | Revenue impact, customer-facing? |
| Technical complexity | High | Dependencies, custom hardware, licensing |
| Cloud readiness | Medium | 12-factor alignment, statelessness |
| Team readiness | Medium | Team's cloud skills, availability |
| Dependency count | High | What else depends on this workload? |

### Typical Wave Sequence

| Wave | Workloads | Purpose |
|------|-----------|---------|
| **Wave 0** | Landing zone, networking, identity, CI/CD | Foundation — no workloads yet |
| **Wave 1** | Non-critical internal tools, dev environments | Build team confidence, validate patterns |
| **Wave 2** | Stateless web apps, APIs, microservices | Low-risk, high-learning workloads |
| **Wave 3** | Stateful services with databases | More complex; validates data migration patterns |
| **Wave 4** | Business-critical production workloads | High-stakes; all patterns proven by now |
| **Wave 5** | Legacy monoliths, mainframe-adjacent systems | Hardest workloads; may need refactoring |

### Dependency Mapping

Before assigning workloads to waves, map their dependencies:

```
App A (Wave 2)
├── Depends on: Database D1 → Must migrate D1 first or in same wave
├── Depends on: Auth Service → Already in cloud (Wave 1) ✓
├── Consumed by: App B → App B migrates in Wave 3 or later
└── Consumed by: App C → App C stays on-prem (hybrid period)
                          → Need cross-environment connectivity
```

---

## Database Migration Patterns

Database migration is the hardest part of any cloud migration. The data must move without loss, and downtime must be minimised.

### Migration Strategy by Database Type

| Source | Target | Strategy |
|--------|--------|----------|
| PostgreSQL on-prem | RDS PostgreSQL / Cloud SQL | Logical replication → cutover |
| Oracle | PostgreSQL (cloud) | Schema conversion + DMS + testing |
| SQL Server | Azure SQL / RDS SQL Server | Native backup/restore or CDC |
| MongoDB | DocumentDB / Atlas / Firestore | mongodump/mongorestore or CDC |
| Custom file-based | S3 / GCS / Blob Storage | Bulk transfer + sync |

### Continuous Replication Pattern

```
Phase 1: Initial Load
┌──────────────────┐     Full dump      ┌──────────────────┐
│ Source Database   │ ──────────────────►│ Target Database   │
│ (on-prem)        │                    │ (cloud)           │
└──────────────────┘                    └──────────────────┘

Phase 2: Change Data Capture (CDC)
┌──────────────────┐     Real-time      ┌──────────────────┐
│ Source Database   │ ── changes ───────►│ Target Database   │
│ (on-prem, active)│                    │ (cloud, replica)  │
└──────────────────┘                    └──────────────────┘
                        Replication lag: seconds to minutes

Phase 3: Cutover
┌──────────────────┐                    ┌──────────────────┐
│ Source Database   │     Stop writes    │ Target Database   │
│ (on-prem)        │ ──────────────────►│ (cloud, active)   │
│                  │     Verify sync     │                  │
│ → Read-only      │     Switch app     │ → Now primary     │
└──────────────────┘                    └──────────────────┘
```

### Schema Migration Checklist

| Item | Check |
|------|-------|
| Data types | All source types map to target equivalents |
| Stored procedures | Converted or replaced with application logic |
| Triggers | Rewritten for target engine or moved to app layer |
| Sequences / auto-increment | Reset correctly after initial load |
| Character encoding | UTF-8 throughout; no encoding mismatches |
| Foreign keys and constraints | Created after initial load, then enabled |
| Indexes | Recreated on target; may need tuning for cloud I/O |

---

## Cutover Strategies

The cutover is the moment you switch production traffic from the old system to the new. Minimising downtime and risk here is the core challenge.

### Cutover Approaches

| Approach | Downtime | Risk | Complexity |
|----------|----------|------|-----------|
| **Big bang** | Hours | High (all-or-nothing) | Low |
| **Blue-green** | Minutes | Medium (instant rollback) | Medium |
| **Canary** | Zero | Low (gradual shift) | High |
| **Strangler fig** | Zero | Lowest (incremental) | Highest |

### Blue-Green Cutover

```
Before cutover:
┌────────────────────┐
│   Load Balancer    │
│   100% → Old Env   │
└────────┬───────────┘
         │
         ▼
┌─────────────────┐      ┌─────────────────┐
│ Old Environment │      │ New Environment  │
│ (on-prem / old) │      │ (cloud / new)    │
│ ← Active        │      │ ← Idle, tested   │
└─────────────────┘      └─────────────────┘

After cutover:
┌────────────────────┐
│   Load Balancer    │
│   100% → New Env   │
└────────┬───────────┘
         │
         ▼
┌─────────────────┐      ┌─────────────────┐
│ Old Environment │      │ New Environment  │
│ (on-prem / old) │      │ (cloud / new)    │
│ ← Standby       │      │ ← Active         │
│   (rollback)     │      │                  │
└─────────────────┘      └─────────────────┘
```

---

## The Strangler Fig Pattern

The strangler fig is the safest pattern for migrating large, complex applications. Instead of rewriting or moving the entire system at once, you incrementally replace functionality.

### How It Works

```
Phase 1: Proxy all traffic through a facade
┌──────────┐     ┌──────────┐     ┌──────────────────┐
│  Client   │────►│  Facade  │────►│  Legacy Monolith │
│           │     │ (API GW) │     │  (100% of routes)│
└──────────┘     └──────────┘     └──────────────────┘

Phase 2: New services handle some routes
┌──────────┐     ┌──────────┐     ┌──────────────────┐
│  Client   │────►│  Facade  │──┬─►│  Legacy Monolith │
│           │     │ (API GW) │  │  │  (80% of routes) │
└──────────┘     └──────────┘  │  └──────────────────┘
                                │
                                └─►┌──────────────────┐
                                   │  New Service A    │
                                   │  (20% of routes)  │
                                   └──────────────────┘

Phase 3: Legacy shrinks over time
┌──────────┐     ┌──────────┐     ┌──────────────────┐
│  Client   │────►│  Facade  │──┬─►│  Legacy (10%)    │
│           │     │ (API GW) │  │  └──────────────────┘
└──────────┘     └──────────┘  ├─►┌──────────────────┐
                                │  │  Service A (30%)  │
                                │  └──────────────────┘
                                ├─►┌──────────────────┐
                                │  │  Service B (35%)  │
                                │  └──────────────────┘
                                └─►┌──────────────────┐
                                   │  Service C (25%)  │
                                   └──────────────────┘

Phase 4: Legacy is decommissioned
```

### Strangler Fig Rules

1. **Never modify the legacy system** — all new features go to the new services
2. **Route at the edge** — the API gateway/load balancer decides which backend handles each request
3. **Migrate data incrementally** — each new service owns its data; sync from legacy during transition
4. **Keep both systems running** — the legacy system is the rollback target until its traffic reaches zero
5. **Celebrate each route migration** — progress is incremental; track percentage of traffic on new vs legacy

---

## Migration Anti-Patterns

| Anti-Pattern | Why It Fails | Alternative |
|-------------|-------------|-------------|
| Big bang migration (everything at once) | Too many moving parts; one failure blocks everything | Migrate in waves, smallest first |
| Refactoring during migration | Two risks at once: new platform + new code | Rehost first, then refactor in the cloud |
| Ignoring data gravity | Moving apps without their data causes cross-network latency | Co-locate compute and data |
| No rollback plan | When the new environment fails, you're stuck | Keep the old environment running until validated |
| Underestimating licensing | Oracle, SQL Server, Windows licences may not transfer | Audit licences before migration planning |
| Skipping the landing zone | Workloads arrive before networking, security, and identity are ready | Wave 0 = infrastructure foundation |

---

## Key Takeaways

- Categorise every workload with the 6 Rs before writing a migration plan — not everything needs to move to the cloud
- Migrate in waves, ordered by dependency and complexity; build team confidence with low-risk workloads first
- Database migration is the hardest part — use continuous replication (CDC) to minimise cutover downtime
- The strangler fig pattern is the safest way to migrate large monoliths: route traffic through a facade and replace incrementally
- Always have a rollback plan: keep the old environment running until the new one is validated in production
- Migration is a business transformation, not a lift-and-shift project — invest in the landing zone (Wave 0) before moving any workloads
