---
title: "Web Security Fundamentals"
weight: 1
---

## Why Web Security Matters

Every web application exposed to the internet is under constant attack. Automated scanners probe for vulnerabilities within minutes of a new service going live. A single SQL injection or misconfigured header can expose millions of user records, destroy a company's reputation, and trigger regulatory fines measured in percentages of global revenue.

Security is not a feature — it is a property of the system. You cannot bolt it on at the end. This section establishes the foundational mental models that underpin every subsequent topic.

---

## The CIA Triad

The three pillars of information security:

| Pillar | Definition | Web Example |
|--------|-----------|-------------|
| **Confidentiality** | Only authorised parties can access data | Encrypting data in transit (TLS), access control on API endpoints |
| **Integrity** | Data cannot be modified without detection | HMAC-signed tokens, database constraints, input validation |
| **Availability** | Systems remain accessible when needed | DDoS protection, rate limiting, redundant infrastructure |

### Extended Models

The CIA triad is the minimum. Real-world security adds:

| Property | Meaning |
|----------|---------|
| **Authentication** | Proving identity (who are you?) |
| **Authorisation** | Proving permission (what can you do?) |
| **Non-repudiation** | Proving an action occurred (audit trails, signatures) |
| **Accountability** | Tracing actions to individuals (logging, monitoring) |

---

## Attack Surfaces

An **attack surface** is the sum of all points where an attacker can attempt to enter or extract data from a system.

### Web Application Attack Surface Map

```text
┌──────────────────────────────────────────────────────┐
│                     INTERNET                         │
└────────────────────────┬─────────────────────────────┘
                         │
            ┌────────────▼────────────┐
            │      CDN / WAF          │  ← Layer 1: Edge
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │    Load Balancer        │  ← Layer 2: Network
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │    Web Server           │  ← Layer 3: HTTP headers, TLS
            │    (Nginx / ALB)        │
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │    Application Code     │  ← Layer 4: Business logic
            │    (Controllers, APIs)  │     Input validation, auth
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │    Data Layer           │  ← Layer 5: Database, cache
            │    (PostgreSQL, Redis)  │     Query construction
            └─────────────────────────┘
```

### Common Entry Points

| Entry Point | Example Attacks |
|-------------|----------------|
| URL path and query parameters | SQL injection, path traversal |
| HTTP headers (`Host`, `Referer`, `User-Agent`) | Header injection, SSRF |
| Cookies | Session hijacking, CSRF |
| Request body (forms, JSON, XML) | XSS, injection, XXE |
| File uploads | Remote code execution, path traversal |
| WebSocket messages | Injection, lack of auth checks |
| Third-party integrations | Supply chain attacks, webhook forgery |

---

## Defence in Depth

No single control stops all attacks. Defence in depth layers multiple independent controls so that a failure in one layer does not compromise the system.

```text
┌─────────────────────────────────────────────┐
│  Layer 1: Network (WAF, firewall, DDoS)     │
│  ┌─────────────────────────────────────────┐ │
│  │  Layer 2: Transport (TLS, HSTS)         │ │
│  │  ┌─────────────────────────────────────┐ │ │
│  │  │  Layer 3: Application (auth, CSP)   │ │ │
│  │  │  ┌─────────────────────────────────┐ │ │ │
│  │  │  │  Layer 4: Data (encryption,     │ │ │ │
│  │  │  │  parameterised queries, hashing)│ │ │ │
│  │  │  └─────────────────────────────────┘ │ │ │
│  │  └─────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### Practical Defence Layers

| Layer | Controls |
|-------|----------|
| **Network** | WAF rules, IP allowlists, DDoS mitigation, VPC segmentation |
| **Transport** | TLS 1.2+, HSTS, certificate transparency |
| **Application** | Input validation, output encoding, CSP, authentication, authorisation |
| **Data** | Encryption at rest (AES-256), parameterised queries, bcrypt/argon2 hashing |
| **Monitoring** | Centralised logging, anomaly detection, alerting, incident response |

---

## The OWASP Top 10 (2021)

The Open Worldwide Application Security Project (OWASP) publishes a regularly updated list of the most critical web application security risks. The 2021 edition:

| # | Category | Description | This Path |
|---|----------|-------------|-----------|
| A01 | **Broken Access Control** | Users act outside intended permissions | Section 5 |
| A02 | **Cryptographic Failures** | Weak or missing encryption of sensitive data | Section 8 |
| A03 | **Injection** | SQL, NoSQL, OS, LDAP injection | Section 2 |
| A04 | **Insecure Design** | Missing security controls at design level | This section |
| A05 | **Security Misconfiguration** | Default credentials, open cloud storage, verbose errors | Section 8, 9 |
| A06 | **Vulnerable & Outdated Components** | Using libraries with known CVEs | Section 10 |
| A07 | **Identification & Authentication Failures** | Broken auth, weak passwords, missing MFA | Section 4 |
| A08 | **Software & Data Integrity Failures** | CI/CD pipeline attacks, unsigned updates | Section 10 |
| A09 | **Security Logging & Monitoring Failures** | No audit trail, no alerting | Section 10 |
| A10 | **Server-Side Request Forgery (SSRF)** | Tricking server into making requests | Section 7 |

### How to Use the OWASP Top 10

The Top 10 is **not** a checklist — it is a risk awareness document. Use it to:

1. **Prioritise** — focus on the categories most relevant to your application
2. **Educate** — use it as a training tool for development teams
3. **Audit** — verify each category is addressed in your threat model
4. **Benchmark** — compare your controls against industry expectations

---

## Threat Modelling

Threat modelling is a structured process for identifying security risks *before* writing code.

### STRIDE Model

| Threat | Definition | Example |
|--------|-----------|---------|
| **S**poofing | Pretending to be another user | Stolen session cookie |
| **T**ampering | Modifying data in transit or at rest | Altering a JWT payload |
| **R**epudiation | Denying an action occurred | No audit logs for admin actions |
| **I**nformation Disclosure | Exposing data to unauthorised parties | Error messages leaking stack traces |
| **D**enial of Service | Making a system unavailable | Sending 10 million requests/second |
| **E**levation of Privilege | Gaining higher permissions | Changing `role=user` to `role=admin` |

### Practical Threat Modelling Steps

1. **Draw the system** — data flow diagrams showing trust boundaries
2. **Identify threats** — apply STRIDE to each component crossing a trust boundary
3. **Rate the risk** — use DREAD or CVSS to prioritise
4. **Mitigate** — design controls for the highest-risk threats
5. **Validate** — test that mitigations actually work

---

## Security Principles

### Principle of Least Privilege

Grant each user, process, and service the minimum permissions required. A read-only API consumer should not have database write access. A Lambda function that reads from S3 should not have `s3:*`.

### Fail Secure

When a system fails, it should deny access rather than allow it. If the authentication service is unavailable, reject all requests — do not silently allow them through.

### Zero Trust

Never assume trust based on network location. Every request must be authenticated and authorised, whether it comes from the internet or from another service inside your VPC.

```text
Traditional:  Internet ──[Firewall]──> Trusted Internal Network
Zero Trust:   Every service ──[AuthN + AuthZ]──> Every other service
```

### Input Validation

All input is untrusted until validated. This includes:

- Query parameters and form fields
- HTTP headers (including `Host`, `Referer`, `X-Forwarded-For`)
- File uploads (name, type, content)
- Data from your own database (it may have been injected earlier)

---

## Security Headers Quick Reference

A preview of headers covered in detail in later sections:

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## Key Takeaways

- **The CIA triad** (Confidentiality, Integrity, Availability) is the foundation of all security thinking — every control maps to one or more of these pillars.
- **Attack surfaces** include every point where data enters or leaves your system — URLs, headers, cookies, file uploads, WebSockets, and third-party integrations.
- **Defence in depth** layers multiple independent controls so that no single failure compromises the system.
- **The OWASP Top 10** is a risk awareness framework, not a checklist — use it to prioritise, educate, and audit.
- **Threat modelling** (STRIDE) identifies security risks at design time, when they are cheapest to fix.
- **Least privilege, fail secure, and zero trust** are non-negotiable principles for any production system.
