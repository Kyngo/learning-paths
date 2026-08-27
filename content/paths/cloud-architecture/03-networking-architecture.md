---
title: "Networking Architecture"
weight: 3
---

# Networking Architecture

Cloud networking is not just "VPC plus subnets." It's the connective tissue between every service, account, region, and on-premises data centre. Mistakes here are expensive to fix because networking underpins everything above it. This section covers the design patterns that scale from one VPC to hundreds of accounts across multiple regions.

---

## VPC Design Patterns

A Virtual Private Cloud (VPC / VNet) is your isolated network within the provider's infrastructure. The design of your VPC determines how workloads communicate, how traffic enters and exits, and where security boundaries lie.

### Standard Three-Tier VPC

```
┌───────────────────────────────── VPC (10.1.0.0/16) ──────────────────────────────────┐
│                                                                                        │
│   ┌──────────── AZ-a ────────────┐    ┌──────────── AZ-b ────────────┐                │
│   │                               │    │                               │                │
│   │  Public Subnet  10.1.1.0/24  │    │  Public Subnet  10.1.4.0/24  │   ← ALB, NAT  │
│   │  ┌─────┐ ┌─────┐            │    │  ┌─────┐ ┌─────┐            │     Gateway     │
│   │  │ ALB │ │ NAT │            │    │  │ ALB │ │ NAT │            │                │
│   │  └─────┘ └─────┘            │    │  └─────┘ └─────┘            │                │
│   │                               │    │                               │                │
│   │  Private Subnet 10.1.2.0/24  │    │  Private Subnet 10.1.5.0/24  │   ← App tier  │
│   │  ┌──────────────────────┐    │    │  ┌──────────────────────┐    │                │
│   │  │  App Containers/VMs  │    │    │  │  App Containers/VMs  │    │                │
│   │  └──────────────────────┘    │    │  └──────────────────────┘    │                │
│   │                               │    │                               │                │
│   │  Data Subnet    10.1.3.0/24  │    │  Data Subnet    10.1.6.0/24  │   ← Databases │
│   │  ┌──────────────────────┐    │    │  ┌──────────────────────┐    │                │
│   │  │  RDS / ElastiCache   │    │    │  │  RDS / ElastiCache   │    │                │
│   │  └──────────────────────┘    │    │  └──────────────────────┘    │                │
│   │                               │    │                               │                │
│   └───────────────────────────────┘    └───────────────────────────────┘                │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Subnet Strategy

| Subnet Type | Internet Access | Use Case |
|------------|----------------|----------|
| **Public** | Inbound + outbound via Internet Gateway | Load balancers, bastion hosts, NAT gateways |
| **Private** | Outbound only via NAT Gateway | Application servers, containers, Lambda in VPC |
| **Isolated** | No internet access | Databases, internal services, sensitive workloads |

### CIDR Planning

CIDR planning is the most common source of regret in cloud networking. Once allocated, CIDRs are hard to change.

**Rules:**

1. **Never overlap** — if VPC A uses 10.1.0.0/16 and VPC B uses 10.1.0.0/16, they cannot be peered
2. **Size generously** — a /16 gives 65,536 IPs; a /24 gives only 254. You will need more than you think
3. **Reserve space** — don't allocate your entire range on day one; leave room for growth
4. **Use an IPAM** — centralised IP Address Management prevents conflicts across hundreds of VPCs

| VPC Size | CIDR | Available IPs | Suitable For |
|----------|------|---------------|-------------|
| Small | /24 | 251 | Single-service, PoC |
| Medium | /20 | 4,091 | Team-level workload |
| Large | /16 | 65,531 | Platform-level, shared services |

### Cross-Provider Comparison

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Virtual network | VPC | VNet | VPC |
| Subnet scope | Per-AZ | Per-region (span AZs) | Per-region (span zones) |
| Route table | Per-subnet | Per-subnet | Per-VPC (or custom) |
| NAT | NAT Gateway (per-AZ) | NAT Gateway | Cloud NAT |
| Internet gateway | IGW (per-VPC) | Implicit for public IPs | Implicit for external IPs |

---

## Hub-and-Spoke Topology

In a multi-account setup, connecting every VPC to every other VPC (full mesh) doesn't scale. Hub-and-spoke centralises connectivity through a single hub.

```
                    ┌──────────────────────┐
                    │     Hub Account       │
                    │   (Transit Gateway)   │
                    │                      │
                    │  ┌────────────────┐  │
     ┌──────────────┤  │  Shared Svcs   │  ├──────────────┐
     │              │  │  DNS, CI/CD    │  │              │
     │              │  └────────────────┘  │              │
     │              │                      │              │
     │              │  ┌────────────────┐  │              │
     │              │  │  Firewall /    │  │              │
     │              │  │  Inspection    │  │              │
     │              │  └────────────────┘  │              │
     │              └──────────┬───────────┘              │
     │                         │                          │
     ▼                         ▼                          ▼
┌─────────┐           ┌─────────────┐            ┌─────────────┐
│ Team A  │           │ Team B      │            │ Team C      │
│ Prod    │           │ Prod        │            │ Prod        │
│ VPC     │           │ VPC         │            │ VPC         │
└─────────┘           └─────────────┘            └─────────────┘
```

### Hub-and-Spoke Benefits

| Benefit | How |
|---------|-----|
| Centralised inspection | All traffic routes through the hub firewall |
| Simplified routing | Spokes only need a route to the hub |
| Shared services | DNS, bastion, CI/CD runners live in the hub |
| On-prem connectivity | VPN or Direct Connect terminates once, in the hub |

### Transit Gateway (AWS) / Azure vWAN / GCP NCC

| Feature | AWS Transit Gateway | Azure Virtual WAN | GCP Network Connectivity Center |
|---------|--------------------|--------------------|-------------------------------|
| Max attachments | 5,000 | 2,000 hubs | Region-based |
| Cross-region | Transit Gateway peering | Global vWAN | Global routing |
| Bandwidth | 50 Gbps per AZ | 20 Gbps per hub | Varies |
| Route tables | Multiple, with isolation | Default + custom | Cloud Router |

---

## Private Connectivity

Not all communication should traverse the public internet — even encrypted. Private connectivity keeps traffic on the provider's backbone or on dedicated physical links.

### Private Link / Private Service Connect

Expose a service in one VPC to consumers in another VPC without VPC peering, without NAT, and without traversing the internet.

```
Producer VPC                          Consumer VPC
┌─────────────────────┐              ┌─────────────────────┐
│                     │              │                     │
│  ┌──────────────┐   │              │  ┌──────────────┐   │
│  │ Service      │   │   Private    │  │ ENI /        │   │
│  │ (NLB / LB)   │◄──┼── Link ─────┼──│ Endpoint     │   │
│  └──────────────┘   │   (provider  │  └──────────────┘   │
│                     │    backbone) │                     │
└─────────────────────┘              └─────────────────────┘
```

**When to use Private Link:**
- Consuming third-party SaaS services without internet exposure
- Cross-account service communication without full VPC peering
- Restricting service access to specific VPCs (not "open to the world")

### Connectivity Options Comparison

| Method | Use Case | Bandwidth | Latency | Cost |
|--------|----------|-----------|---------|------|
| VPC Peering | Two VPCs, same or cross-region | 50 Gbps+ | Low | Per-GB transfer |
| Transit Gateway | Many VPCs, hub-and-spoke | 50 Gbps/AZ | Low | Hourly + per-GB |
| Private Link | Service exposure without peering | Service-dependent | Low | Hourly + per-GB |
| VPN (IPsec) | On-prem to cloud, encrypted | 1.25 Gbps per tunnel | Variable | Hourly |
| Direct Connect / ExpressRoute | Dedicated physical link | 1–100 Gbps | Consistent | Monthly + per-GB |

---

## DNS Architecture

DNS in a multi-account cloud environment is more complex than a single hosted zone. You need to resolve:
- Public names (customer-facing)
- Private names (service-to-service within the cloud)
- On-premises names (hybrid resolution)
- Cross-account names (shared services)

### DNS Resolution Flow

```
Application in Team A VPC
        │
        │ Resolves: api.internal.company.com
        ▼
VPC DNS Resolver (e.g., Route 53 Resolver / Cloud DNS)
        │
        ├── Is it a private hosted zone associated with this VPC?
        │   └── Yes → return private IP
        │
        ├── Is there a forwarding rule to on-prem?
        │   └── Yes → forward to on-prem DNS (via VPN/DX)
        │
        └── Fall through to public DNS
            └── Resolve via recursive resolver → public IP
```

### Split-Horizon DNS

Same domain name resolves differently depending on where the query originates:

| Query Source | `api.company.com` Resolves To |
|-------------|-------------------------------|
| Internet (customer) | Public IP (load balancer) |
| Internal VPC | Private IP (internal LB) |
| On-premises | Private IP via DNS forwarding |

---

## Hybrid Connectivity

Connecting on-premises to cloud securely and reliably.

### Decision Framework

```
Do you need > 1 Gbps bandwidth?
├── Yes
│   └── Do you need consistent, low latency?
│       ├── Yes → Direct Connect / ExpressRoute (dedicated)
│       └── No  → Direct Connect / ExpressRoute (hosted)
└── No
    └── Is traffic intermittent?
        ├── Yes → Site-to-site VPN (on-demand)
        └── No  → Site-to-site VPN (persistent) or hosted connection
```

### Architecture with Redundancy

```
On-Premises Data Centre
┌─────────────────────────────────┐
│  Router A ──────┐               │
│                  │               │
│  Router B ──────┤               │
└─────────────────┼───────────────┘
                   │
    ┌──────────────┼───────────────┐
    │ Primary DX   │  Backup VPN   │
    │ (1 Gbps)     │  (IPsec)      │
    └──────┬───────┴──────┬────────┘
           │              │
           ▼              ▼
    ┌──────────────────────────────┐
    │      Transit Gateway          │
    │      (Hub Account)            │
    └──────┬──────────────┬────────┘
           │              │
     ┌─────▼────┐   ┌────▼─────┐
     │ Prod VPC │   │ Dev VPC  │
     └──────────┘   └──────────┘
```

**Key design rules:**
- Always have a backup path (VPN as backup for Direct Connect)
- Terminate hybrid connections in the hub account, not individual workload accounts
- Use BGP for dynamic route propagation between on-prem and cloud
- Encrypt everything, even over Direct Connect (Direct Connect is private, not encrypted)

---

## Key Takeaways

- Design VPCs with three tiers (public, private, isolated) and plan CIDRs generously using centralised IPAM
- Hub-and-spoke topology scales better than full mesh — centralise inspection, routing, and shared services in the hub
- Private connectivity (Private Link, peering) keeps traffic off the public internet and reduces attack surface
- DNS architecture in multi-account environments requires split-horizon, cross-account zone sharing, and hybrid forwarding
- Hybrid connectivity needs redundancy: Direct Connect for primary bandwidth, VPN for failover
- CIDR conflicts are the most common networking regret — plan the address space before creating the first VPC
