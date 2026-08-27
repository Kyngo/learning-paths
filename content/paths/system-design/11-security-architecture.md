---
title: "Security Architecture"
weight: 11
---

# Security Architecture

Security is not a feature you bolt on after launch — it is an architectural concern woven through every layer of the system. This section covers the patterns for authentication at scale, zero-trust networking, and secrets management in distributed systems.

---

## Authentication at Scale

### Token-Based Authentication (JWT)

```
Client → Login(credentials) → Auth Service → JWT ← Client stores token
Client → API Request + JWT → API Gateway → Validates JWT → Backend Service
```

| Component | Responsibility |
|-----------|---------------|
| Auth Service | Validates credentials, issues tokens |
| JWT (access token) | Short-lived (5-15 min), carries claims (user ID, roles) |
| Refresh token | Long-lived (days/weeks), used to get new access tokens |
| API Gateway | Validates JWT signature, checks expiration, extracts claims |

### OAuth 2.0 / OpenID Connect Flows

| Flow | Use Case | Example |
|------|----------|---------|
| Authorization Code + PKCE | Web apps, SPAs, mobile | "Log in with Google" |
| Client Credentials | Service-to-service | Backend calling another backend |
| Device Authorization | Smart TVs, CLIs | "Enter this code at example.com/device" |

### Session vs Token

| | Server-Side Sessions | JWT Tokens |
|-|---------------------|-----------|
| Storage | Server (Redis, DB) | Client (cookie, localStorage) |
| Revocation | Delete session record | Cannot revoke until expiry (unless denylist) |
| Scaling | Requires shared session store | Stateless — any server can validate |
| Size | Session ID (small) | JWT payload (larger, contains claims) |

---

## Authorisation Patterns

### RBAC (Role-Based Access Control)

```
User → has → Role → has → Permissions
Alice → Admin → [read, write, delete]
Bob → Editor → [read, write]
Carol → Viewer → [read]
```

Simple, works for most applications. Breaks down when permissions depend on the specific resource.

### ABAC (Attribute-Based Access Control)

Decisions based on attributes of the user, resource, and environment:

```
ALLOW if:
  user.department == resource.department
  AND user.clearance >= resource.classification
  AND environment.time is within business hours
```

More flexible than RBAC but harder to audit and reason about.

### ReBAC (Relationship-Based Access Control)

Permissions based on relationships in a graph (used by Google Zanzibar, AuthZed, Ory):

```
document:budget#viewer@user:alice    ← Alice can view the budget document
folder:finance#editor@group:finance  ← Finance group can edit the finance folder
document:budget#parent@folder:finance ← Budget inherits folder permissions
```

Best for: Google Docs-style sharing, hierarchical permissions, complex access graphs.

---

## Zero-Trust Architecture

**Principle:** Never trust, always verify. Every request is authenticated and authorised regardless of network location.

| Traditional (Perimeter) | Zero-Trust |
|------------------------|-----------|
| Trust inside the firewall | Trust nothing |
| VPN = access | Identity = access |
| Flat internal network | Microsegmented |
| Implicit trust between services | mTLS between all services |

### Implementation

| Layer | Mechanism |
|-------|-----------|
| Service identity | mTLS certificates (SPIFFE/SPIRE) |
| Service-to-service auth | JWT or mTLS |
| Network segmentation | Service mesh (Istio, Linkerd), security groups |
| API gateway | Rate limiting, WAF, JWT validation, IP filtering |
| User identity | SSO + MFA at every entry point |

---

## Secrets Management

### The Problem

Applications need secrets (database passwords, API keys, encryption keys). Hardcoding them is a security disaster. Environment variables are better but not enough.

### Solutions

| Approach | Tool | How It Works |
|----------|------|-------------|
| Vault | HashiCorp Vault | Centralised secret store, dynamic credentials, lease/revocation |
| Cloud-native | AWS Secrets Manager, GCP Secret Manager | Managed, rotation, IAM-integrated |
| Config injection | Kubernetes Secrets, ECS task definition | Injected at deploy time |
| Parameter store | AWS SSM Parameter Store | Hierarchical, versioned, encrypted |

### Secret Rotation Pattern

```
1. Vault generates new DB credentials
2. Application fetches new credentials (or lease is renewed)
3. Old credentials are revoked after grace period
4. No human ever sees the password
```

### Encryption at Rest and in Transit

| Layer | Mechanism |
|-------|-----------|
| In transit | TLS 1.3 (everywhere), mTLS (service-to-service) |
| At rest | AES-256 (S3 SSE, RDS encryption, EBS encryption) |
| Application-level | Envelope encryption (data key encrypted by master key) |
| Key management | AWS KMS, GCP Cloud KMS, HashiCorp Vault |

---

## API Gateway as Security Layer

| Concern | Implementation |
|---------|---------------|
| Authentication | JWT validation, API key verification |
| Rate limiting | Per-user, per-IP, per-endpoint |
| WAF | OWASP Top 10 protection, SQL injection, XSS |
| IP filtering | Allow/deny lists, geo-blocking |
| Request validation | Schema validation, payload size limits |
| TLS termination | Offload TLS from backend services |
| Logging | Audit trail of all requests |

---

## Key Takeaways

- JWT for stateless auth at scale, sessions for simpler apps with revocation needs.
- OAuth 2.0 + PKCE is the standard for web/mobile. Client Credentials for service-to-service.
- RBAC for most apps, ABAC for policy-rich environments, ReBAC for document-sharing models.
- Zero-trust: mTLS between services, identity-based access, no implicit trust from network location.
- Never hardcode secrets. Use a vault or managed secret store with automatic rotation.
- The API gateway is your security perimeter: auth, rate limiting, WAF, validation — all in one place.
