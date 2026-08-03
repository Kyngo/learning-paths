---
title: "Architecture Decisions"
weight: 8
---

# Architecture Decisions

Every software system is shaped by hundreds of decisions — technology choices, structural patterns, integration approaches. The best teams make these decisions **explicitly, traceably, and reversibly**. Architecture Decision Records (ADRs) are the primary tool for this discipline.

---

## Architecture Decision Records (ADRs)

An ADR documents **one significant architectural decision** along with its context, rationale, and consequences.

### Standard Format

```markdown
# ADR-001: Use PostgreSQL for order storage

## Status
Accepted (2024-03-15)

## Context
The orders service needs persistent storage supporting:
- ACID transactions for financial data
- Complex queries for reporting
- High write throughput (10K orders/hour peak)

We considered PostgreSQL, DynamoDB, and MongoDB.

## Decision
We will use PostgreSQL (via AWS RDS).

## Consequences
### Positive
- Strong consistency guarantees
- Rich query capabilities (JOIN, aggregates)
- Team expertise exists

### Negative
- Vertical scaling limits (vs DynamoDB)
- Operational overhead (backups, failover)
- Schema migrations required for changes

### Risks
- May need read replicas at 100K orders/hour
```

### ADR Lifecycle

| Status | Meaning |
|--------|---------|
| **Proposed** | Under discussion, not yet approved |
| **Accepted** | Approved and active |
| **Deprecated** | No longer recommended but existing code may use it |
| **Superseded** | Replaced by another ADR (link to successor) |

### Where to Store ADRs

| Location | Pros | Cons |
|----------|------|------|
| Repository (`docs/adr/`) | Versioned with code, close to context | Scattered across repos |
| Wiki/Confluence | Searchable, cross-project visibility | Not versioned with code |
| Dedicated ADR repo | Central registry, easy to browse | Disconnected from implementation |

> **Recommendation:** Store in the repo for service-specific decisions; use a central wiki for cross-cutting decisions.

---

## Decision Criteria

Not all decisions deserve equal rigor. Evaluate decisions on three axes:

### Reversibility

| Category | Examples | Process |
|----------|----------|---------|
| **One-way door** (irreversible) | Database choice, public API contract, cloud provider | Full ADR, team review, stakeholder alignment |
| **Two-way door** (reversible) | Library choice, internal data format, CI tool | Lightweight ADR, team discussion |
| **Trivial** | Variable naming, folder structure | Just decide, no ADR needed |

### Blast Radius

| Scope | Impact | Governance |
|-------|--------|------------|
| Single service | Only one team affected | Team decides autonomously |
| Multiple services | Cross-team coordination needed | Architecture review + ADR |
| Organization-wide | Standards, platforms, security | Architecture board approval |

### Alignment

Decisions should align with:

- **Business strategy** — Does this support where the company is going?
- **Technical strategy** — Is this consistent with the tech radar?
- **Team capabilities** — Can the team build and maintain this?
- **Existing decisions** — Does this contradict prior ADRs?

---

## Fitness Functions

**Fitness functions** are automated checks that validate whether the system still meets its architectural goals over time.

### Types

| Type | Validates | Example |
|------|-----------|---------|
| **Atomic** | Single characteristic | P95 latency < 200ms |
| **Holistic** | Combined characteristics | Availability AND cost within budget |
| **Triggered** | On event (deploy, PR) | No circular dependencies in build |
| **Continuous** | Always running | SLO dashboards, chaos tests |

### Examples

```yaml
# Deployment fitness function
- name: "No service takes > 15 minutes to deploy"
  metric: deployment_duration_p95
  threshold: "< 900s"
  frequency: per_deployment

# Coupling fitness function  
- name: "No service has > 3 synchronous dependencies"
  check: dependency_graph_analysis
  threshold: "max_sync_deps <= 3"
  frequency: weekly

# Security fitness function
- name: "No critical CVEs in production"
  check: vulnerability_scan
  threshold: "critical_count == 0"
  frequency: daily
```

### Architecture Tests (Code-Level)

```java
// ArchUnit (Java) — enforce layering
@ArchTest
static final ArchRule layerRule = layeredArchitecture()
    .layer("Controller").definedBy("..controller..")
    .layer("Service").definedBy("..service..")
    .layer("Repository").definedBy("..repository..")
    .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
    .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
    .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service");
```

```python
# Python — enforce no circular imports
# deptry / import-linter
[importlinter]
root_package = myapp
[importlinter:contract:layers]
name = Layered architecture
type = layers
layers =
    myapp.api
    myapp.service
    myapp.repository
```

---

## Evolutionary Architecture

Architecture should evolve incrementally rather than be designed upfront for all future needs.

### Principles

| Principle | Meaning |
|-----------|---------|
| **Last responsible moment** | Defer decisions until you have the most information |
| **Small, reversible steps** | Prefer many small changes over large irreversible ones |
| **Fitness functions** | Automate validation of architectural properties |
| **Sacrificial architecture** | Accept that today's design will be replaced |

### Guided Evolution

```
              ┌─────────────────────────────────────────┐
              │  Fitness Functions (automated guardrails) │
              └─────────────────┬───────────────────────┘
                                │
    Start ──→ [Simple] ──→ [Modular] ──→ [Distributed] ──→ [Adaptive]
                │               │              │                │
            Monolith      Modular mono    Microservices    Event-driven
```

### Avoiding Evolutionary Traps

| Trap | Consequence | Prevention |
|------|-------------|------------|
| Premature abstraction | Complexity without value | Wait for the third use case |
| Accidental coupling | Can't evolve independently | Fitness functions for coupling |
| Missing tests | Can't refactor safely | Test coverage as fitness function |
| Knowledge silos | Decisions lost when people leave | ADRs + architecture reviews |

---

## Tech Radar

A **tech radar** communicates the organization's technology strategy:

| Ring | Meaning | Action |
|------|---------|--------|
| **Adopt** | Proven, recommended for broad use | Default choice for new projects |
| **Trial** | Worth exploring in low-risk projects | Use with awareness of immaturity |
| **Assess** | Interesting, worth investigating | Research and prototype only |
| **Hold** | Do not start new work with this | Migrate away when practical |

### Example Entries

| Technology | Ring | Rationale |
|------------|------|-----------|
| PostgreSQL | Adopt | Proven, team expertise, managed service available |
| io_uring | Trial | Performance gains demonstrated; limited production experience |
| CockroachDB | Assess | Interesting for multi-region; evaluate operational costs |
| MongoDB (unstructured data) | Hold | Past issues with data consistency; prefer PostgreSQL JSONB |

### Maintaining the Radar

- Review quarterly with engineering leads
- Any engineer can propose additions/movements
- Decisions to move items require lightweight ADR
- Publish openly — transparency drives adoption

---

## Architecture Reviews

### When to Review

| Trigger | Review Type |
|---------|-------------|
| New service or major feature | Design review (before implementation) |
| ADR with org-wide blast radius | Architecture board review |
| Post-incident (architectural root cause) | Post-mortem architecture review |
| Quarterly | Health check / fitness function review |

### Lightweight Architecture Review

```markdown
## Review Checklist

### Structural
- [ ] Clear bounded context / responsibility
- [ ] Dependencies explicit and minimal
- [ ] No circular dependencies
- [ ] Data ownership clear

### Quality Attributes
- [ ] Scalability approach documented
- [ ] Failure modes identified
- [ ] Performance targets defined
- [ ] Security model documented

### Evolutionary
- [ ] Decision reversibility assessed
- [ ] ADR written for significant decisions
- [ ] Fitness functions defined
- [ ] Migration path from current state clear
```

---

## Communicating Decisions

Decisions are worthless if the team doesn't know about them.

### Communication Channels

| Audience | Channel | Format |
|----------|---------|--------|
| Implementing team | ADR in repo + team meeting | Technical detail |
| Adjacent teams | Architecture guild + shared wiki | Context + impact |
| Leadership | Architecture review deck | Business value + risks |
| New joiners | Onboarding docs referencing ADRs | Narrative + links |

### Decision Announcement Template

```markdown
## 📐 Architecture Decision: [Title]

**What:** [One-line summary of the decision]
**Why:** [Business/technical driver in plain language]
**Impact:** [Who needs to change behavior and how]
**ADR:** [Link to full ADR]
**Questions:** [Contact person or channel]
```

### Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Decision by committee | Slow, watered-down choices | Clear decision maker (RACI) |
| Not documenting context | Future team reverses good decisions | Always write the "why" |
| Documenting after the fact | Missing rationale and alternatives | Write ADR during discussion |
| Over-governing | Teams slow to ship | Only govern high-blast-radius decisions |

---

## Key Takeaways

1. **Write ADRs** for any decision that is hard to reverse or affects multiple teams.
2. Classify decisions by **reversibility and blast radius** — govern accordingly.
3. **Fitness functions** automate architectural compliance — don't rely on manual reviews alone.
4. Architecture should **evolve incrementally** — avoid big upfront design.
5. A **tech radar** communicates strategy; keep it alive with quarterly reviews.
6. **Communicate decisions** through multiple channels — an ADR nobody reads is worthless.
7. The goal is not perfect architecture but **informed, traceable, reversible** decisions.
