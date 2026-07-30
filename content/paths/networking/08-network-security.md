---
title: "Network Security"
weight: 8
---

## Defense in Depth

Network security uses multiple layers — if one fails, others still protect:

```mermaid
flowchart TD
    Internet["Internet"]
    Internet --> Edge["Edge: WAF, DDoS protection"]
    Edge --> FW["Perimeter: Firewall"]
    FW --> SG["Host: Security Groups / iptables"]
    SG --> App["Application: Auth, input validation"]
    App --> Data["Data: Encryption at rest"]
```

---

## Firewalls

### Types

| Type | Layer | What It Inspects | Example |
|------|-------|-----------------|---------|
| Packet filter | 3-4 | IP, port, protocol | iptables, ACLs |
| Stateful | 3-4 | Connection state + packets | AWS Security Groups |
| Application (WAF) | 7 | HTTP content, patterns | AWS WAF, Cloudflare |
| Next-gen (NGFW) | 3-7 | Deep packet inspection | Palo Alto, Fortinet |

### Stateful vs Stateless

| Aspect | Stateless (NACLs) | Stateful (Security Groups) |
|--------|-------------------|---------------------------|
| Tracks connections | No | Yes |
| Return traffic | Must explicitly allow | Automatically allowed |
| Rule evaluation | All rules, in order | All rules, most permissive wins |
| Use case | Subnet-level control | Instance-level control |

### Security Group Example (AWS)

```text
Inbound Rules:
  Allow TCP 443 from 0.0.0.0/0        (HTTPS from anywhere)
  Allow TCP 22 from 10.0.0.0/16       (SSH from VPC only)

Outbound Rules:
  Allow All traffic to 0.0.0.0/0       (default — allow all outbound)
```

**Principle of least privilege:** Only open ports that are needed, only from sources that need access.

---

## NAT (Network Address Translation)

Allows private IPs to communicate with the internet through a shared public IP:

```mermaid
flowchart LR
    subgraph Private["Private Network"]
        A["10.0.1.10"]
        B["10.0.1.11"]
        C["10.0.1.12"]
    end
    
    Private --> NAT["NAT Gateway<br/>203.0.113.5"]
    NAT --> Internet["Internet"]
```

### NAT Types

| Type | Direction | Use Case |
|------|-----------|----------|
| SNAT (Source NAT) | Outbound | Private hosts access internet |
| DNAT (Destination NAT) | Inbound | Port forwarding to internal servers |
| PAT (Port Address Translation) | Outbound | Many-to-one (most common) |

In AWS: **NAT Gateway** provides SNAT for private subnets.

---

## VPN (Virtual Private Network)

Encrypted tunnel over public internet — extends a private network:

```mermaid
flowchart LR
    Office["Office<br/>10.0.1.0/24"] <-->|"Encrypted Tunnel<br/>(over Internet)"| Cloud["AWS VPC<br/>10.0.2.0/24"]
```

### VPN Types

| Type | Use Case | Protocol |
|------|----------|----------|
| Site-to-Site | Connect two networks | IPsec |
| Client VPN | Remote worker access | OpenVPN, WireGuard |
| AWS VPN | Connect on-premises to VPC | IPsec over Internet |
| Direct Connect | Dedicated private link (not VPN) | Physical fiber |

---

## TLS/SSL

Encrypts data in transit between client and server (covered in detail in HTTP/HTTPS section).

### mTLS (Mutual TLS)

Both sides present certificates — used for service-to-service communication:

```mermaid
sequenceDiagram
    participant A as Service A
    participant B as Service B
    
    A->>B: ClientHello + Client Certificate
    B->>A: ServerHello + Server Certificate
    Note over A,B: Both verify each other's identity
    Note over A,B: Encrypted channel established
```

Use cases: microservice mesh, API gateways, zero-trust networks.

---

## Common Attack Types

| Attack | Layer | Description | Mitigation |
|--------|-------|-------------|-----------|
| DDoS | 3-7 | Overwhelm with traffic | WAF, rate limiting, CDN |
| SYN flood | 4 | Exhaust TCP connection table | SYN cookies, rate limiting |
| DNS amplification | 7 | Reflect/amplify via DNS | Rate limiting, BCP38 |
| Man-in-the-middle | 4-7 | Intercept communications | TLS, certificate pinning |
| Port scanning | 4 | Discover open services | Firewall, IDS |
| ARP spoofing | 2 | Redirect local traffic | DAI, static ARP |

---

## Network Segmentation

Isolate workloads to limit blast radius:

```mermaid
flowchart TD
    subgraph Public["Public Subnet (DMZ)"]
        LB["Load Balancer"]
    end
    subgraph Private["Private Subnet"]
        App["App Servers"]
    end
    subgraph Data["Data Subnet"]
        DB["Databases"]
    end
    
    Internet --> LB
    LB --> App
    App --> DB
    DB -.-x Internet
```

| Zone | Access | Contains |
|------|--------|----------|
| Public (DMZ) | Internet-facing | Load balancers, bastion hosts |
| Private | Internal only | Application servers |
| Data | Highly restricted | Databases, caches |

---

## Zero Trust

Traditional: "Trust everything inside the network perimeter."
Zero Trust: "Never trust, always verify."

| Principle | Implementation |
|-----------|---------------|
| Verify identity | mTLS, OAuth tokens for every request |
| Least privilege | Minimal permissions per service |
| Assume breach | Encrypt everything, log everything |
| Micro-segmentation | Per-service security groups |

---

## Key Takeaways

1. **Defense in depth** — multiple layers, not a single firewall
2. **Least privilege** — only open what's needed, from where it's needed
3. **Stateful firewalls** (Security Groups) track connections automatically
4. **NAT** hides private IPs behind a public IP
5. **TLS everywhere** — encrypt all traffic, even internal
6. **Segment networks** — public, private, data zones with strict boundaries
7. **Zero trust** — verify every request regardless of network location
