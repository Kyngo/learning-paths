---
title: "Secure Software Development"
weight: 10
---

## Why This Matters

Most vulnerabilities are introduced during development. Fixing a vulnerability in production costs 30–100x more than fixing it during design. Secure development integrates security into every phase of the SDLC — shifting left means finding and fixing issues earlier, when they're cheapest.

---

## Secure SDLC

```mermaid
flowchart LR
    Req["Requirements<br/>· Security Requirements<br/>· Abuse Cases"] --> Design["Design<br/>· Threat Model<br/>· Architecture Review"]
    Design --> Code["Code<br/>· Secure Coding<br/>· SAST + SCA"]
    Code --> Test["Test<br/>· DAST + Pentest<br/>· Fuzz Testing"]
    Test --> Deploy["Deploy<br/>· Config Review<br/>· Secrets Check"]
    Deploy --> Operate["Operate<br/>· Monitor<br/>· Respond + Patch"]
```

### Security Activities per Phase

| Phase | Security Activities | Output |
|-------|-------------------|--------|
| Requirements | Security requirements, abuse cases, compliance needs | Security user stories |
| Design | Threat modeling, architecture review, crypto design | Threat model document |
| Implementation | Secure coding standards, SAST, code review | Clean code, no critical findings |
| Testing | DAST, penetration testing, fuzz testing | Security test report |
| Deployment | Configuration review, secrets scanning, hardening | Secure deployment |
| Operations | Monitoring, vulnerability management, incident response | Ongoing security posture |

---

## Threat Modeling

### STRIDE Model

Systematic identification of threats per component:

| Threat | Against | Question | Mitigation |
|--------|---------|----------|-----------|
| **S**poofing | Authentication | Can someone pretend to be another user? | Strong auth, MFA |
| **T**ampering | Integrity | Can data be modified in transit/at rest? | Signatures, checksums, TLS |
| **R**epudiation | Non-repudiation | Can someone deny their actions? | Audit logs, digital signatures |
| **I**nformation Disclosure | Confidentiality | Can data be exposed? | Encryption, access controls |
| **D**enial of Service | Availability | Can the system be made unavailable? | Rate limiting, scaling, redundancy |
| **E**levation of Privilege | Authorization | Can someone gain unauthorized access? | Least privilege, input validation |

### Threat Modeling Process

```mermaid
flowchart TD
    Scope["1: Define Scope<br/>(what are we modeling?)"]
    Scope --> Diagram["2: Create Data Flow Diagram<br/>(components, data flows, trust boundaries)"]
    Diagram --> Identify["3: Identify Threats<br/>(STRIDE per element)"]
    Identify --> Mitigate["4: Define Mitigations<br/>(controls per threat)"]
    Mitigate --> Validate["5: Validate<br/>(are mitigations implemented?)"]
```

### Data Flow Diagram Elements

| Element | Symbol | Example |
|---------|--------|---------|
| Process | Circle | Web server, API service |
| Data Store | Parallel lines | Database, file system |
| External Entity | Rectangle | User, third-party API |
| Data Flow | Arrow | HTTP request, DB query |
| Trust Boundary | Dashed line | Internet/DMZ, DMZ/internal |

---

## Secure Coding Practices

### Input Validation

| Rule | Implementation |
|------|---------------|
| Validate all input | Server-side validation (never trust client) |
| Allowlist over denylist | Define what's valid, reject everything else |
| Validate type, length, range | Reject before processing |
| Canonicalize before validation | Decode, normalize, then validate |
| Reject on failure | Don't try to "fix" bad input |

### Output Encoding

| Context | Encoding | Prevents |
|---------|----------|----------|
| HTML body | HTML entity encoding | XSS |
| HTML attribute | Attribute encoding | XSS |
| JavaScript | JavaScript encoding | XSS |
| URL parameter | URL encoding | Injection |
| SQL | Parameterized queries | SQL injection |
| OS command | Avoid entirely; use APIs | Command injection |

### Authentication & Session

| Practice | Implementation |
|----------|---------------|
| Hash passwords properly | Argon2id or bcrypt (never MD5/SHA) |
| Enforce password complexity | Minimum length (12+), check against breached lists |
| Implement account lockout | Temporary lockout after N failures |
| Secure session tokens | Cryptographically random, sufficient length |
| Set cookie flags | `Secure; HttpOnly; SameSite=Strict` |
| Implement proper logout | Invalidate server-side session |

### Error Handling

| Do | Don't |
|----|-------|
| Log detailed errors server-side | Expose stack traces to users |
| Return generic error messages | Reveal internal paths or versions |
| Use structured error codes | Leak database schema in errors |
| Fail securely (deny access on error) | Fail open (grant access on error) |

---

## Security Testing

### SAST (Static Application Security Testing)

Analyzes source code without executing it:

| Tool | Languages | Integration |
|------|-----------|-------------|
| SonarQube | 30+ languages | CI/CD, IDE |
| Semgrep | 30+ languages | CI/CD, pre-commit |
| CodeQL | Major languages | GitHub Actions |
| Checkmarx | Enterprise | CI/CD |
| Bandit | Python | CI/CD, pre-commit |

**Strengths:** Finds code-level flaws early, covers all code paths.
**Weaknesses:** High false positive rate, no runtime context.

### DAST (Dynamic Application Security Testing)

Tests the running application from outside:

| Tool | Type | Use Case |
|------|------|----------|
| OWASP ZAP | Open source | CI/CD integration, API testing |
| Burp Suite | Commercial | Manual + automated testing |
| Nuclei | Open source | Template-based scanning |

**Strengths:** Finds runtime issues, low false positives.
**Weaknesses:** Only tests reachable code paths, slower.

### SCA (Software Composition Analysis)

Identifies vulnerable dependencies:

| Tool | Coverage | Features |
|------|----------|----------|
| Snyk | Multi-language | Fix PRs, license compliance |
| Dependabot | GitHub native | Auto-update PRs |
| Trivy | Containers + code | Fast, comprehensive |
| OWASP Dependency-Check | Java, .NET | Free, CI/CD |
| Renovate | Multi-platform | Configurable auto-updates |

### Fuzz Testing

Sends random/malformed input to find crashes and unexpected behavior:

| Type | Method | Finds |
|------|--------|-------|
| Dumb fuzzing | Random mutations | Buffer overflows, crashes |
| Smart fuzzing | Grammar-aware mutations | Logic errors, parsing bugs |
| Coverage-guided | Maximize code coverage | Deep bugs in complex code |

---

## Supply Chain Security

### Attack Vectors

```mermaid
flowchart TD
    SC["Supply Chain Attacks"]
    SC --> Dep["Dependency Attacks"]
    SC --> Build["Build Pipeline Attacks"]
    SC --> Dist["Distribution Attacks"]
    
    Dep --> Typo["Typosquatting<br/>(lodash → lodahs)"]
    Dep --> Takeover["Maintainer Takeover<br/>(compromised npm account)"]
    Dep --> Malicious["Malicious Update<br/>(event-stream incident)"]
    
    Build --> CI["CI/CD Compromise<br/>(stolen secrets)"]
    Build --> Inject["Build Injection<br/>(SolarWinds)"]
    
    Dist --> Tamper["Package Tampering<br/>(modified binary)"]
    Dist --> Mirror["Compromised Mirror<br/>(fake repository)"]
```

### Defenses

| Defense | Mechanism |
|---------|-----------|
| Lock files | Pin exact versions (package-lock.json, Pipfile.lock) |
| Integrity verification | Check hashes/signatures of packages |
| Private registry | Mirror approved packages internally |
| SBOM generation | Know exactly what's in your software |
| Dependency review | Review new dependencies before adding |
| Automated updates | Renovate/Dependabot with CI gates |
| SLSA framework | Supply chain integrity levels |

### SLSA (Supply-chain Levels for Software Artifacts)

| Level | Requirements |
|-------|-------------|
| SLSA 1 | Build process documented |
| SLSA 2 | Hosted build service, authenticated provenance |
| SLSA 3 | Hardened build platform, non-falsifiable provenance |
| SLSA 4 | Two-person review, hermetic builds |

---

## Secrets Management in Code

### What NOT to Do

```text
# NEVER commit these:
API_KEY=sk-abc123def456
DATABASE_URL=postgres://admin:password@prod-db:5432/app
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### Secrets Detection

| Tool | Integration | Features |
|------|-------------|----------|
| git-secrets | Pre-commit hook | AWS-focused patterns |
| truffleHog | CI/CD, pre-commit | Entropy + regex detection |
| Gitleaks | CI/CD, pre-commit | Fast, configurable rules |
| GitHub Secret Scanning | GitHub native | Alerts on pushed secrets |

### Secrets Management Solutions

| Solution | Use Case |
|----------|----------|
| AWS Secrets Manager | Application secrets, auto-rotation |
| HashiCorp Vault | Multi-cloud, dynamic secrets |
| Environment variables | Simple deployments (with caution) |
| CI/CD variables | Pipeline secrets (masked, protected) |

---

## Security in CI/CD Pipeline

```mermaid
flowchart LR
    Commit["Commit"] --> Secrets["Secrets<br/>Scan"]
    Secrets --> SAST["SAST"]
    SAST --> SCA["SCA<br/>(dependencies)"]
    SCA --> Build["Build"]
    Build --> Image["Container<br/>Scan"]
    Image --> DAST["DAST<br/>(staging)"]
    DAST --> Gate{"Security<br/>Gate"}
    Gate -->|Pass| Deploy["Deploy"]
    Gate -->|Fail| Block["Block +<br/>Notify"]
```

### Security Gates

| Gate | Blocks On | Rationale |
|------|-----------|-----------|
| Secrets scan | Any detected secret | Prevents credential leaks |
| SAST | Critical/High findings | Prevents code-level vulnerabilities |
| SCA | Critical CVEs, license violations | Prevents known vulnerable dependencies |
| Container scan | Critical CVEs in base image | Prevents deploying vulnerable containers |
| DAST | Critical findings | Prevents runtime vulnerabilities |

---

## Key Takeaways

1. **Shift left** — security in requirements and design, not just testing
2. **Threat model every significant feature** — STRIDE gives structure
3. **Automate in CI/CD** — SAST, SCA, secrets scanning on every commit
4. **Supply chain is a real threat** — lock versions, verify integrity, use SBOM
5. **Never commit secrets** — use pre-commit hooks + secrets managers
6. **Security gates prevent deployment of known-bad code** — block on critical
7. **Defense in depth applies to code too** — validate input, encode output, fail securely
