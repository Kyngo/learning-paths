---
title: "Multi-Account & Organisation Strategy"
weight: 2
---

# Multi-Account & Organisation Strategy

A single cloud account for everything is the number one architectural mistake organisations make. It creates a blast radius that spans your entire business: one misconfigured IAM policy can expose production data, development experiments can exhaust production quotas, and cost attribution becomes impossible. This section covers how to structure accounts, enforce guardrails, and automate provisioning.

---

## Why Multiple Accounts?

A single account conflates environments, teams, and workloads into one security and billing boundary. Multiple accounts provide:

| Benefit | What It Solves |
|---------|---------------|
| **Blast radius reduction** | A compromise in dev cannot reach prod |
| **Independent quotas** | Dev load testing doesn't exhaust prod API limits |
| **Cost allocation** | Each account maps to a cost centre or team |
| **Least privilege** | Developers get admin in sandbox, read-only in prod |
| **Compliance boundaries** | PCI/HIPAA workloads isolated in dedicated accounts |
| **Autonomous teams** | Teams own their accounts without blocking each other |

---

## Account Structure Patterns

### The Organisational Unit (OU) Hierarchy

Every cloud provider organises accounts into a tree. The tree structure enforces policies that inherit downward.

```
Organisation Root
├── Security OU
│   ├── Log Archive Account          ← Centralised logs (immutable)
│   ├── Security Tooling Account     ← GuardDuty, Security Hub, SIEM
│   └── Audit Account                ← Read-only cross-account access
├── Infrastructure OU
│   ├── Networking Account           ← Transit Gateway, DNS, VPN
│   ├── Shared Services Account      ← CI/CD, artefact repos, container registry
│   └── Identity Account             ← SSO, identity provider federation
├── Workloads OU
│   ├── Production OU
│   │   ├── Team A Prod Account
│   │   ├── Team B Prod Account
│   │   └── Team C Prod Account
│   ├── Pre-Production OU
│   │   ├── Team A Pre Account
│   │   ├── Team B Pre Account
│   │   └── Team C Pre Account
│   └── Development OU
│       ├── Team A Dev Account
│       ├── Team B Dev Account
│       └── Team C Dev Account
├── Sandbox OU
│   ├── Developer Sandbox 1          ← Experimentation, auto-deleted
│   └── Developer Sandbox 2
└── Suspended OU                     ← Decommissioned accounts (quarantine)
```

### Account-per-Environment vs Account-per-Team

| Strategy | When to Use | Trade-Off |
|----------|------------|-----------|
| Account per environment | Small org, few services | Simple but limited isolation between teams |
| Account per team per environment | Medium org, autonomous teams | Better isolation, more accounts to manage |
| Account per workload per environment | Large org, strict compliance | Maximum isolation, highest management overhead |

**Recommendation for most organisations:** Account per team per environment. One team gets three accounts (dev, pre, prod). This balances isolation with manageability.

---

## Service Control Policies (SCPs)

SCPs are guardrails applied at the OU or account level that restrict what *any* identity in the account can do — even the root user.

### SCP Design Principles

1. **Deny, don't allow** — SCPs are most effective as deny lists on top of IAM allow policies
2. **Apply to OUs, not individual accounts** — reduces drift
3. **Test in sandbox first** — a wrong SCP can lock everyone out
4. **Always allow the breakglass role** — exempt your emergency access role from SCPs

### Common SCP Patterns

| SCP | Purpose | Applied To |
|-----|---------|-----------|
| Deny region outside allowed list | Data sovereignty | All OUs |
| Deny leaving the organisation | Prevent account theft | All OUs |
| Deny disabling CloudTrail/logging | Audit integrity | All OUs |
| Deny creating IAM users (force SSO) | Identity hygiene | Workloads OU |
| Deny public S3 buckets | Data leak prevention | Production OU |
| Deny expensive instance types | Cost control | Sandbox OU |
| Allow all (no restrictions) | Experimentation | Sandbox OU (with budget alert) |

### SCP Inheritance

```
Organisation Root (Deny leave org, Deny disable logging)
│
├── Production OU (Deny public S3, Deny delete backups)
│   │
│   └── Team A Prod Account
│       → Inherits: Deny leave org + Deny disable logging
│                  + Deny public S3 + Deny delete backups
│
└── Sandbox OU (Deny expensive instances, Budget: £100/month)
    │
    └── Sandbox Account
        → Inherits: Deny leave org + Deny disable logging
                   + Deny expensive instances
```

---

## Landing Zones

A landing zone is the pre-configured, multi-account environment that provides the foundation for all cloud workloads. Think of it as the "factory floor" on which teams build.

### Landing Zone Components

| Component | Purpose |
|-----------|---------|
| Account factory | Automated account creation with standard config |
| Identity federation | SSO from corporate IdP to all accounts |
| Networking baseline | Transit networking, DNS, VPN/Direct Connect |
| Logging pipeline | Centralised CloudTrail, VPC flow logs, DNS logs |
| Security baseline | GuardDuty, Config rules, Security Hub enabled |
| Cost management | Budgets, anomaly detection, cost allocation tags |
| Guardrails | SCPs and preventive controls |

### Provider Implementations

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Organisation | AWS Organizations | Management Groups | Google Cloud Organisation |
| Landing zone tool | Control Tower | Azure Landing Zones (CAF) | Cloud Foundation Toolkit |
| Account factory | Account Factory for Terraform (AFT) | Subscription vending | Project Factory |
| Guardrails | SCPs + Config Rules | Azure Policy | Organisation Policies |
| SSO | IAM Identity Center | Entra ID | Cloud Identity |
| Centralised logging | CloudTrail → S3 (log archive) | Activity Logs → Log Analytics | Audit Logs → BigQuery |

### Landing Zone Architecture (Cloud-Agnostic View)

```
┌─────────────────────────────────────────────────────┐
│                   MANAGEMENT PLANE                    │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Identity &   │  │ Policy &     │  │ Cost       │ │
│  │ SSO          │  │ Guardrails   │  │ Management │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
├─────────────────────────────────────────────────────┤
│                   SECURITY PLANE                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Centralised  │  │ Security     │  │ Audit &    │ │
│  │ Logging      │  │ Monitoring   │  │ Compliance │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
├─────────────────────────────────────────────────────┤
│                   NETWORK PLANE                       │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Transit Hub  │  │ DNS          │  │ Hybrid     │ │
│  │              │  │ Resolution   │  │ Connect    │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
├─────────────────────────────────────────────────────┤
│                   WORKLOAD PLANE                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Team A       │  │ Team B       │  │ Team C     │ │
│  │ (dev/pre/    │  │ (dev/pre/    │  │ (dev/pre/  │ │
│  │  prod)       │  │  prod)       │  │  prod)     │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## Account Vending

Account vending is the automated process of creating new accounts with all the baseline configurations pre-applied.

### What a Vended Account Includes

1. **Account creation** in the organisation under the correct OU
2. **Networking** — VPC with standard CIDR from the IPAM pool, connected to the transit hub
3. **Identity** — SSO roles provisioned (developer, admin, read-only)
4. **Security** — logging enabled, security tools deployed, SCPs inherited
5. **Cost** — budget alerts configured, cost allocation tags enforced
6. **Bootstrapping** — IaC state backend (S3 + DynamoDB / GCS / Azure Storage) created

### Account Vending Flow

```
Developer requests account (via portal or IaC PR)
        │
        ▼
Account Factory (Terraform/CloudFormation/Bicep)
        │
        ├── 1. Create account in organisation
        ├── 2. Move to correct OU
        ├── 3. Apply SCP guardrails
        ├── 4. Deploy networking baseline
        ├── 5. Configure SSO roles
        ├── 6. Enable security tooling
        ├── 7. Set budget alerts
        └── 8. Notify team (Slack/email)
        │
        ▼
Account ready — team can deploy workloads
```

### Self-Service Guardrails

The goal is: **developers move fast, guardrails prevent mistakes.**

| Permission | Sandbox | Dev | Pre-Prod | Prod |
|-----------|---------|-----|----------|------|
| Create any resource | ✅ | ✅ | ✅ | Via pipeline only |
| Delete resources | ✅ | ✅ | With approval | Via pipeline only |
| Access console | ✅ | ✅ | Read-only | Break-glass only |
| Direct SSH/RDP | ✅ | Via bastion | Via bastion | Denied |
| Modify IAM | ✅ | Limited | Denied | Denied |
| Budget limit | £100/mo | £500/mo | Shared | Shared |

---

## Identity Federation

Centralised identity is non-negotiable in a multi-account setup. Users should not have IAM users in each account — they should authenticate once through a corporate identity provider and assume roles in target accounts.

### Federation Architecture

```
Corporate IdP (Okta, Entra ID, Google Workspace)
        │
        │ SAML 2.0 / OIDC
        ▼
Cloud SSO Service (IAM Identity Center / Entra ID / Cloud Identity)
        │
        │ Assume Role
        ├────────────────────────────────────┐
        ▼                                    ▼
  Team A Prod Account                 Team B Dev Account
  (Role: ReadOnly)                    (Role: PowerUser)
```

### Key Decisions

| Decision | Recommendation |
|----------|---------------|
| IdP choice | Use your existing corporate IdP — don't create a second one |
| Role granularity | 3–5 roles per account (admin, developer, read-only, CI/CD, break-glass) |
| Session duration | 1 hour for humans, 12 hours for CI/CD pipelines |
| MFA enforcement | Required for all human access, everywhere, always |
| Service accounts | Use workload identity (IRSA, Workload Identity Federation) — not long-lived keys |

---

## Tagging Strategy

Tags are the metadata layer that makes cost allocation, automation, and compliance work across hundreds of accounts.

### Mandatory Tags

| Tag Key | Purpose | Example Values |
|---------|---------|---------------|
| `Environment` | Cost split, policy enforcement | `dev`, `pre`, `prod`, `sandbox` |
| `Team` | Ownership and accountability | `platform`, `payments`, `search` |
| `CostCentre` | Finance attribution | `CC-1234`, `ENG-001` |
| `Application` | Service identification | `booking-api`, `search-indexer` |
| `ManagedBy` | Drift detection | `terraform`, `manual`, `cdk` |

### Enforcement

Tags are useless if they're optional. Enforce them with:

1. **SCP/Policy** — deny resource creation without mandatory tags
2. **IaC validation** — pre-commit hooks and CI checks for tag presence
3. **Config rules** — detect and flag untagged resources retroactively
4. **Cost anomaly alerts** — untagged resources trigger immediate alerts

---

## Key Takeaways

- Use multiple accounts to reduce blast radius, isolate environments, enforce quotas, and attribute costs
- Structure accounts into OUs that reflect your organisational boundaries (security, infrastructure, workloads, sandbox)
- SCPs are guardrails, not permissions — they restrict what's possible even for admin users
- A landing zone provides the pre-configured foundation: identity, networking, logging, security, and cost controls
- Account vending automates new account creation so teams self-serve without waiting weeks
- Federate identity from your corporate IdP — never create IAM users in workload accounts
- Enforce a tagging strategy from day one; retrofitting tags across hundreds of accounts is painful
