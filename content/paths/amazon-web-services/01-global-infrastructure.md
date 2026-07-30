---
title: "AWS Global Infrastructure"
weight: 1
---

## The Physical Layer

AWS operates the world's largest cloud infrastructure — a global network of data centers organized into a hierarchy designed for high availability, low latency, and data sovereignty.

```mermaid
flowchart TD
    Global["AWS Global Infrastructure"]
    Global --> Regions["33 Regions<br>(geographic areas)"]
    Global --> Edge["450+ Edge Locations<br>(CDN endpoints)"]
    
    Regions --> AZs["105+ Availability Zones<br>(isolated data centers)"]
    Regions --> Local["Local Zones<br>(city extensions)"]
    Regions --> Wavelength["Wavelength Zones<br>(5G edge)"]
```

---

## Regions

A Region is a geographic area containing multiple isolated data centers. Each region is completely independent — a failure in one region doesn't affect others.

### Region Naming

| Region Code | Location | Common Use |
|-------------|----------|-----------|
| `us-east-1` | N. Virginia | Default for many services, largest |
| `eu-central-1` | Frankfurt | European workloads, GDPR |
| `eu-west-1` | Ireland | European workloads |
| `ap-southeast-1` | Singapore | Asia-Pacific |
| `us-west-2` | Oregon | US West Coast |

### Choosing a Region

```mermaid
flowchart TD
    Start["Choose a Region"] --> Compliance{"Data residency<br>requirements?"}
    Compliance -->|"Yes"| Restricted["Use required region<br>(e.g., EU for GDPR)"]
    Compliance -->|"No"| Latency{"Where are<br>your users?"}
    Latency --> Closest["Pick closest region"]
    Closest --> Services{"All services<br>available?"}
    Services -->|"No"| Alternative["Pick next closest<br>with all services"]
    Services -->|"Yes"| Cost{"Cost<br>sensitive?"}
    Cost -->|"Yes"| Compare["Compare pricing<br>(us-east-1 often cheapest)"]
    Cost -->|"No"| Done["✅ Use selected region"]
```

**Key factors:**

1. **Compliance** — GDPR requires EU data residency; some industries require specific countries
2. **Latency** — closer to users = faster response times
3. **Service availability** — new services launch in us-east-1 first
4. **Cost** — same service can cost 10-20% more in some regions

---

## Availability Zones (AZs)

Each region has 3-6 AZs. Each AZ is one or more physically separate data centers with independent power, cooling, and networking.

```mermaid
flowchart LR
    subgraph Region["eu-central-1 (Frankfurt)"]
        AZ1["AZ 1a<br>Data Center(s)"]
        AZ2["AZ 1b<br>Data Center(s)"]
        AZ3["AZ 1c<br>Data Center(s)"]
    end
    
    AZ1 <-->|"< 2ms latency<br>redundant fiber"| AZ2
    AZ2 <-->|"< 2ms latency<br>redundant fiber"| AZ3
    AZ1 <-->|"< 2ms latency<br>redundant fiber"| AZ3
```

### AZ Properties

- **Physically isolated** — separate buildings, power grids, flood zones
- **Connected** — high-bandwidth, low-latency private fiber between AZs
- **Independent failure** — a power outage in AZ-a doesn't affect AZ-b
- **Mapped differently per account** — your `eu-central-1a` might be a different physical AZ than another account's `eu-central-1a`

### Multi-AZ Architecture

Deploying across multiple AZs is the foundation of high availability on AWS:

```mermaid
flowchart TD
    Users["Users"] --> ALB["Application Load Balancer<br>(spans all AZs)"]
    
    subgraph AZ1["AZ 1a"]
        EC2a["EC2 Instance"]
        RDSp["RDS Primary"]
    end
    
    subgraph AZ2["AZ 1b"]
        EC2b["EC2 Instance"]
        RDSs["RDS Standby<br>(synchronous replica)"]
    end
    
    ALB --> EC2a
    ALB --> EC2b
    RDSp -.->|"Sync replication"| RDSs
```

If AZ-a fails: ALB routes all traffic to AZ-b, RDS fails over to standby.

---

## Edge Locations

Edge locations are AWS's CDN (CloudFront) and DNS (Route 53) endpoints distributed globally — 450+ locations in 90+ cities.

| Service | Uses Edge Locations For |
|---------|------------------------|
| **CloudFront** | Caching static/dynamic content close to users |
| **Route 53** | DNS resolution at the nearest location |
| **AWS WAF** | Filtering malicious traffic at the edge |
| **Lambda@Edge** | Running code at CDN edge |
| **Global Accelerator** | Routing to optimal endpoint via AWS backbone |

---

## AWS Backbone Network

AWS operates its own global fiber network connecting all regions and edge locations. Traffic between regions travels over AWS's private network, not the public internet.

Benefits:

- **Lower latency** — optimized routing
- **Higher throughput** — dedicated capacity
- **Better security** — traffic never touches public internet
- **Consistent performance** — no ISP congestion

---

## Service Scope

Services operate at different scopes:

| Scope | Examples | Implication |
|-------|----------|-------------|
| **Global** | IAM, Route 53, CloudFront, WAF | One instance worldwide |
| **Regional** | VPC, S3, Lambda, ECS, RDS | Must be created per region |
| **AZ-level** | EC2, EBS, Subnets | Tied to a specific AZ |

```mermaid
flowchart TD
    subgraph Global["Global Services"]
        IAM["IAM"]
        R53["Route 53"]
        CF["CloudFront"]
    end
    
    subgraph Regional["Regional Services (eu-central-1)"]
        S3["S3"]
        Lambda["Lambda"]
        VPC["VPC"]
    end
    
    subgraph AZ["AZ-Scoped (eu-central-1a)"]
        EC2["EC2"]
        EBS["EBS Volume"]
        Subnet["Subnet"]
    end
```

Understanding scope matters for:

- **Disaster recovery** — regional services need cross-region replication
- **High availability** — AZ-scoped resources need multi-AZ deployment
- **Cost** — cross-region data transfer costs money

---

## Shared Responsibility Model

```mermaid
flowchart TD
    subgraph Customer["Customer Responsibility (Security IN the Cloud)"]
        CData["Customer Data"]
        CApp["Application & IAM"]
        COS["OS, Network, Firewall Config"]
        CEnc["Client-side Encryption"]
    end
    
    subgraph AWS["AWS Responsibility (Security OF the Cloud)"]
        AHW["Hardware / Global Infrastructure"]
        ANet["Networking"]
        AVirt["Virtualization (Hypervisor)"]
        APhys["Physical Security"]
    end
```

| AWS Manages | You Manage |
|-------------|-----------|
| Physical data centers | Your data and encryption |
| Hardware and networking | IAM users, roles, policies |
| Hypervisor and host OS | Security groups and NACLs |
| Managed service patching | Application code and dependencies |
| Global infrastructure | OS patching (EC2) |

---

## Key Takeaways

1. **Regions are independent** — choose based on compliance, latency, services, and cost
2. **AZs provide high availability** — always deploy across 2+ AZs for production
3. **Edge locations reduce latency** — use CloudFront for static content
4. **Know service scope** — global vs regional vs AZ determines your HA strategy
5. **Shared responsibility** — AWS secures the infrastructure; you secure what you put on it
6. **AWS backbone** — inter-region traffic stays on AWS's private network
