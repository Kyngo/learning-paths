---
title: Cloud Architecture Patterns
weight: 111
bookCollapseSection: true
---

# Cloud Architecture Patterns

Cloud-agnostic architecture patterns and strategies that transfer across providers. This path teaches the principles behind well-designed cloud systems — not the vendor-specific services that implement them. Use it alongside the [AWS]({{< relref "/paths/amazon-web-services" >}}), [Terraform]({{< relref "/paths/terraform" >}}), and [System Design]({{< relref "/paths/system-design" >}}) paths to connect theory to practice.

## Prerequisites

This path assumes familiarity with:

- **System Design** fundamentals (scalability, load balancing, caching) — covered in the [System Design]({{< relref "/paths/system-design" >}}) path
- **Networking basics** (IP, DNS, HTTP, TCP) — covered in the [Networking]({{< relref "/paths/networking" >}}) path
- **Microservice patterns** (saga, service mesh, API gateway) — covered in [System Design: Microservices]({{< relref "/paths/system-design/07-microservices" >}})
- **Reliability patterns** (circuit breaker, retry, bulkhead) — covered in [System Design: Reliability & Resilience]({{< relref "/paths/system-design/09-reliability-resilience" >}})
- **AWS Well-Architected Framework** — covered in [AWS: Well-Architected]({{< relref "/paths/amazon-web-services/13-well-architected" >}})
- **FinOps and AWS cost management** — covered in [AWS: Cost Management]({{< relref "/paths/amazon-web-services/15-cost-management" >}})

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Cloud Computing Fundamentals]({{< relref "01-cloud-computing-fundamentals" >}}) | Service models (IaaS/PaaS/SaaS/FaaS), shared responsibility, cloud-native principles, 12-factor app, cloud vs on-prem decisions |
| 2 | [Multi-Account & Organisation Strategy]({{< relref "02-multi-account-organisation-strategy" >}}) | Account structure, OUs, SCPs, landing zones, Control Tower, Azure Landing Zones, GCP org structure, account vending, identity federation |
| 3 | [Networking Architecture]({{< relref "03-networking-architecture" >}}) | VPC design, hub-and-spoke, transit gateway, PrivateLink, VPC peering, Direct Connect/ExpressRoute, DNS architecture, hybrid connectivity |
| 4 | [Compute Patterns]({{< relref "04-compute-patterns" >}}) | VM vs containers vs serverless, auto-scaling patterns, spot/preemptible strategies, right-sizing methodology, GPU/ML compute |
| 5 | [Data Architecture Patterns]({{< relref "05-data-architecture-patterns" >}}) | Polyglot persistence, data gravity, data mesh, data sovereignty, storage tiering, backup/archive, cross-region replication |
| 6 | [Cloud Migration Strategies]({{< relref "06-cloud-migration-strategies" >}}) | 6 Rs, migration wave planning, database migration, cutover strategies, strangler fig, legacy modernisation |
| 7 | [Serverless Architecture]({{< relref "07-serverless-architecture" >}}) | Event-driven serverless, cold starts, function composition, step functions/workflows, serverless databases, anti-patterns |
| 8 | [Multi-Cloud & Hybrid Patterns]({{< relref "08-multi-cloud-hybrid-patterns" >}}) | When multi-cloud fits, abstraction layers, cross-cloud service mesh, data sync, vendor lock-in analysis, cloud-agnostic IaC |
| 9 | [Security Architecture]({{< relref "09-security-architecture" >}}) | Zero-trust, identity-based perimeter, encryption patterns, secrets management, compliance automation, security as code |
| 10 | [Cost Architecture]({{< relref "10-cost-architecture" >}}) | Cost-aware design, architecture trade-offs, reserved capacity, spot patterns, cost allocation, FinOps team structure, unit economics |

## What You'll Be Able To Do

- Design cloud systems using transferable patterns that work on any provider
- Structure multi-account organisations with proper isolation and governance
- Make informed compute, data, and networking decisions based on workload characteristics
- Plan and execute cloud migrations with minimal downtime
- Evaluate multi-cloud strategies critically and avoid unnecessary complexity
- Build cost-aware architectures from day one
