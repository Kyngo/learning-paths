---
title: "Security Architecture"
weight: 9
---

# Security Architecture

Cloud security is fundamentally different from on-premises security. There's no physical perimeter to defend. Instead, identity becomes the perimeter, encryption becomes the default, and automation replaces manual audit. This section covers the architectural patterns for securing cloud environments without slowing down delivery.

---

## Zero-Trust in the Cloud

Zero-trust is not a product — it's an architecture principle: **never trust, always verify.** Every request is authenticated and authorised, regardless of its origin.

### Traditional vs Zero-Trust

```
Traditional (Perimeter-Based)             Zero-Trust (Identity-Based)
┌────────────────────────────┐           ┌────────────────────────────┐
│         TRUSTED ZONE       │           │  Every request verified:   │
│                            │           │                            │
│  ┌────┐  ┌────┐  ┌────┐  │           │  ┌────┐  ┌────┐  ┌────┐  │
│  │Svc │──│Svc │──│Svc │  │           │  │Svc │─?│Svc │─?│Svc │  │
│  │ A  │  │ B  │  │ C  │  │           │  │ A  │  │ B  │  │ C  │  │
│  └────┘  └────┘  └────┘  │           │  └────┘  └────┘  └────┘  │
│                            │           │  ? = authn + authz + mTLS │
│  "Inside = trusted"       │           │  "Nothing is trusted"     │
│  (one breach = full access)│           │  (breach is contained)    │
└────────────────────────────┘           └────────────────────────────┘
```

### Zero-Trust Principles

| Principle | Implementation |
|-----------|---------------|
| **Verify explicitly** | Authenticate every request (user, service, device) |
| **Least privilege** | Grant minimum permissions needed, with time limits |
| **Assume breach** | Segment networks, encrypt everything, monitor continuously |
| **No implicit trust** | Internal network ≠ trusted; treat VPC traffic as potentially hostile |
| **Continuous validation** | Re-evaluate access decisions based on context (location, device, behaviour) |

---

## Identity-Based Perimeter

In the cloud, the network is no longer the security boundary — identity is. Every request carries an identity, and every resource has a policy.

### Identity Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Identity Provider (IdP)                     │
│                   (Okta, Entra ID, Google Workspace)             │
└──────────┬──────────────────────────────┬───────────────────────┘
           │ OIDC / SAML                  │ Workload Identity
           ▼                              ▼
┌──────────────────────┐       ┌──────────────────────────────┐
│  Human Access         │       │  Machine/Service Access       │
│                      │       │                              │
│  SSO → Assume Role   │       │  IRSA / Workload Identity    │
│  MFA enforced        │       │  Federation                  │
│  Session: 1 hour     │       │  No long-lived keys          │
│  Audited             │       │  Short-lived tokens           │
└──────────────────────┘       └──────────────────────────────┘
```

### Service-to-Service Authentication

| Pattern | Security | Complexity | Use When |
|---------|----------|-----------|----------|
| **IAM roles (IRSA, workload identity)** | High | Low | Service within same cloud |
| **mTLS (mutual TLS)** | Very high | Medium | Cross-service, service mesh |
| **API keys** | Low | Low | Legacy integration (avoid for new) |
| **OAuth 2.0 client credentials** | High | Medium | Cross-org API access |
| **Short-lived tokens (STS)** | High | Low | Temporary cross-account access |

### The IAM Decision Hierarchy

```
Can you use the cloud provider's identity? (IRSA, workload identity)
├── Yes → Use it. No keys, automatic rotation, audit trail built in.
└── No
    ├── Can you use mTLS? (service mesh, internal services)
    │   └── Yes → Use it. Identity tied to certificate.
    └── No
        ├── Can you use OAuth 2.0 with short-lived tokens?
        │   └── Yes → Use it. Machine-to-machine flow.
        └── Last resort → API key, but:
            ├── Rotate frequently (90 days max)
            ├── Store in secrets manager
            └── Never embed in code or config files
```

---

## Encryption Patterns

### Encryption at Rest

| Approach | Who Manages Key | Key Visibility | Use Case |
|----------|----------------|---------------|----------|
| Provider-managed (SSE) | Provider | No visibility | Default for most workloads |
| Customer-managed key (CMK) | You (via KMS) | Full control, audit | Regulated data, compliance |
| Client-side encryption | You (app layer) | Full control | Maximum security, zero-trust of provider |

### Encryption in Transit

| Layer | Mechanism | Notes |
|-------|-----------|-------|
| External (internet) | TLS 1.2+ with managed certificates | ACM, Let's Encrypt, managed SSL |
| Internal (VPC) | TLS between services or mTLS via service mesh | Not encrypted by default on most clouds |
| Database connections | TLS with certificate pinning | Enforce `sslmode=verify-full` |
| Storage API calls | HTTPS (enforced via bucket policy) | Deny `http://` requests to S3/GCS |

### Encryption in Use (Confidential Computing)

Emerging pattern where data is encrypted even while being processed:

| Technology | Provider | Use Case |
|-----------|----------|----------|
| Nitro Enclaves | AWS | Processing sensitive data in isolated enclaves |
| Confidential VMs | GCP, Azure | Memory encrypted by hardware (SEV-SNP, TDX) |
| Clean Rooms | AWS, Snowflake | Multi-party computation without exposing raw data |

---

## Secrets Management

Secrets (API keys, database passwords, certificates) must never be in code, config files, or environment variables baked into images.

### Secrets Architecture

```
┌──────────────────────────────────────────────┐
│               Secrets Manager                  │
│         (Vault, AWS SM, GCP SM, AKV)          │
│                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ DB pass  │  │ API keys │  │ TLS certs│   │
│  │ (auto-   │  │ (rotated │  │ (auto-   │   │
│  │  rotated)│  │  90 days)│  │  renewed)│   │
│  └──────────┘  └──────────┘  └──────────┘   │
└───────────────────────┬──────────────────────┘
                        │ IAM-authenticated request
                        ▼
            ┌──────────────────────┐
            │  Application         │
            │  (fetches at startup │
            │   or via sidecar)    │
            └──────────────────────┘
```

### Secrets Management Rules

| Rule | Implementation |
|------|---------------|
| No secrets in code | Pre-commit hooks scan for secrets (gitleaks, truffleHog) |
| No secrets in CI variables | Use secrets manager, referenced by name |
| Automatic rotation | Secrets Manager rotation Lambda / function |
| Least privilege access | Only the consuming service has `secretsmanager:GetSecretValue` |
| Audit all access | CloudTrail / audit logs for every `GetSecret` call |
| No shared secrets | Each service/environment gets its own secret |

---

## Compliance Automation

Manual compliance audits are slow, expensive, and point-in-time. Cloud-native compliance is continuous, automated, and evidence-generating.

### Compliance as Code Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Compliance Pipeline                         │
│                                                               │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────┐  │
│  │ Policy as     │──►│ Continuous     │──►│ Evidence     │  │
│  │ Code          │   │ Evaluation     │   │ Collection   │  │
│  │               │   │                │   │              │  │
│  │ (OPA, Config  │   │ (Config Rules, │   │ (Automated   │  │
│  │  Rules, Azure │   │  Security Hub, │   │  reports,    │  │
│  │  Policy)      │   │  SCC)          │   │  dashboards) │  │
│  └──────────────┘   └────────────────┘   └──────────────┘  │
│                                                               │
│  Detect → Alert → Auto-remediate (or block)                  │
└─────────────────────────────────────────────────────────────┘
```

### Common Compliance Controls

| Control | Implementation | Automation |
|---------|---------------|-----------|
| Encryption at rest | Default encryption on all storage | Config rule: deny unencrypted resources |
| Logging enabled | CloudTrail / audit logs in all accounts | SCP: deny disabling CloudTrail |
| Public access blocked | No public S3 buckets, no public IPs without approval | Account-level S3 public access block |
| MFA enforced | SSO with MFA; deny console without MFA | IAM policy condition: `aws:MultiFactorAuthPresent` |
| Tagging compliance | Required tags on all resources | Config rule + auto-tag remediation |
| Vulnerability scanning | Container image scanning in CI/CD | Block deployment if critical CVEs found |

---

## Security as Code

Security controls should be version-controlled, peer-reviewed, tested, and deployed through the same pipeline as application code.

### What Gets Codified

| Security Domain | Code Artefact |
|----------------|---------------|
| Network rules | Security groups, NACLs (Terraform) |
| IAM policies | Policy documents (Terraform/CDK/Bicep) |
| Encryption config | KMS keys, bucket policies (Terraform) |
| Compliance rules | OPA/Rego policies, Config rules |
| Incident response | Lambda/Cloud Function auto-remediation |
| Secret rotation | Rotation Lambda triggered by Secrets Manager |
| Vulnerability gates | CI/CD pipeline stages (Trivy, Snyk, SonarQube) |

### Security in the CI/CD Pipeline

```
Code Push
    │
    ▼
┌──────────────┐
│ Static Analysis │  ← SAST (SonarQube, Semgrep)
│ Secret Scanning │  ← gitleaks, truffleHog
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Dependency    │  ← SCA (Snyk, Dependabot, Trivy)
│ Scanning      │     CVE database check
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Container     │  ← Image scanning (Trivy, ECR scan)
│ Scanning      │     Base image CVEs, misconfigurations
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ IaC Scanning  │  ← tfsec, checkov, cfn-nag
│               │     Misconfigured resources
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Deploy        │  ← Only if all gates pass
└──────────────┘
```

---

## Cloud-Native SIEM

Traditional SIEMs are on-premises appliances designed for network logs. Cloud-native SIEM aggregates and correlates cloud-specific signals.

### Signal Sources

| Source | What It Captures |
|--------|-----------------|
| Cloud audit logs (CloudTrail, Activity Log) | API calls, who did what |
| VPC flow logs | Network connections, traffic patterns |
| DNS query logs | Resolution attempts (including malicious domains) |
| Application logs | Business logic errors, auth failures |
| Container runtime | Process execution, file system changes |
| Identity events | Login attempts, MFA failures, privilege escalation |
| Config changes | Resource creation, modification, deletion |

### Detection Architecture

```
Signal Sources                  Analysis                     Response
┌───────────────┐              ┌──────────────┐            ┌──────────────┐
│ CloudTrail    │─┐            │              │            │ Alert →      │
│ VPC Flow Logs │─┤            │ Correlation  │            │ Slack/Pager  │
│ DNS Logs      │─┼───────────►│ Engine       │───────────►│              │
│ GuardDuty     │─┤            │ (SIEM /      │            │ Auto-        │
│ App Logs      │─┘            │  Security    │            │ Remediate    │
│               │              │  Lake)       │            │ (Lambda /    │
└───────────────┘              └──────────────┘            │  Step Fn)    │
                                                            └──────────────┘
```

---

## Key Takeaways

- Zero-trust treats every request as untrusted regardless of network origin — identity is the new perimeter
- Prefer cloud-native workload identity (IRSA, workload identity federation) over long-lived API keys
- Encrypt at rest by default (provider-managed keys); use customer-managed keys when regulation demands it
- Centralise secrets in a secrets manager with automatic rotation — never in code, CI variables, or config files
- Automate compliance with policy-as-code (OPA, Config Rules, Azure Policy) and integrate security scanning into CI/CD
- Cloud-native SIEM correlates audit logs, flow logs, DNS logs, and identity events to detect threats automatically
- Security controls are code: version-controlled, peer-reviewed, tested, and deployed through pipelines
