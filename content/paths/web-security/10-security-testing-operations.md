---
title: "Security Testing & Operations"
weight: 10
---

## Security in the Development Lifecycle

Security testing is not a one-time gate before release. It is a continuous process woven into every stage of the software development lifecycle — from the first line of code to production monitoring.

```text
┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌───────────┐
│  Code    │──▶│  Build   │──▶│  Test    │──▶│  Deploy  │──▶│  Monitor  │
│          │   │          │   │          │   │          │   │           │
│  SAST    │   │  SCA     │   │  DAST    │   │  Headers │   │  WAF logs │
│  Linting │   │  License │   │  Pentest │   │  TLS     │   │  Alerting │
│  Secrets │   │  SBOM    │   │  Fuzzing │   │  Config  │   │  Incident │
└─────────┘   └──────────┘   └──────────┘   └──────────┘   └───────────┘
```

---

## SAST (Static Application Security Testing)

SAST analyses source code (or bytecode) without executing it. It finds vulnerabilities in the code itself — injection flaws, hardcoded secrets, insecure configurations.

### How SAST Works

```text
Source code → Parser → Abstract Syntax Tree → Taint analysis → Vulnerability report
                                                │
                                     Tracks user input (sources)
                                     through code paths (propagators)
                                     to dangerous operations (sinks)
```

### Common SAST Tools

| Tool | Languages | Open Source | Notes |
|------|-----------|-------------|-------|
| **Semgrep** | Python, Java, JS, Go, Ruby, etc. | ✅ (community rules) | Pattern-based, easy custom rules |
| **SonarQube** | 30+ languages | ✅ (Community Edition) | Comprehensive, integrates with CI |
| **CodeQL** | Java, JS, Python, Go, C/C++, Ruby | ✅ (GitHub) | Semantic analysis, powerful queries |
| **Bandit** | Python only | ✅ | Fast, Python-specific |
| **SpotBugs + Find Security Bugs** | Java only | ✅ | Bytecode analysis |

### Semgrep Example

```yaml
# .semgrep.yml — custom rule: detect SQL string concatenation
rules:
  - id: sql-string-concat
    patterns:
      - pattern: |
          $QUERY = f"... {$INPUT} ..."
      - metavariable-regex:
          metavariable: $QUERY
          regex: (?i)(select|insert|update|delete)
    message: "Possible SQL injection via string formatting"
    languages: [python]
    severity: ERROR
```

Run:

```bash
semgrep --config .semgrep.yml --config p/owasp-top-ten src/
```

### CI Integration (GitLab CI)

```yaml
sast:
  stage: test
  image: returntocorp/semgrep
  script:
    - semgrep --config p/owasp-top-ten --config p/python --json --output semgrep-results.json src/
  artifacts:
    reports:
      sast: semgrep-results.json
  allow_failure: false
```

---

## DAST (Dynamic Application Security Testing)

DAST tests the running application by sending malicious requests and observing responses. It finds vulnerabilities that SAST cannot — misconfigurations, authentication issues, business logic flaws.

### How DAST Works

```text
DAST scanner → Crawls application (discovers endpoints)
             → Sends attack payloads (injection, XSS, etc.)
             → Analyses responses (error messages, timing, content)
             → Reports vulnerabilities
```

### Common DAST Tools

| Tool | Type | Open Source | Notes |
|------|------|-------------|-------|
| **OWASP ZAP** | Proxy / scanner | ✅ | Industry standard, extensible |
| **Nuclei** | Template-based scanner | ✅ | Fast, community templates |
| **Burp Suite** | Proxy / scanner | ❌ (free Community Edition) | Professional penetration testing |
| **Nikto** | Web server scanner | ✅ | Server misconfigurations |

### OWASP ZAP in CI

```bash
# Run ZAP baseline scan against a staging environment
docker run --rm \
  -v "$(pwd)/zap-report:/zap/wrk:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
    -t https://staging.example.com \
    -r zap-report.html \
    -c zap-rules.conf
```

### Nuclei Template Scan

```bash
# Scan for known vulnerabilities and misconfigurations
nuclei -u https://staging.example.com \
  -t http/technologies/ \
  -t http/misconfiguration/ \
  -t http/vulnerabilities/ \
  -severity critical,high \
  -o nuclei-results.txt
```

---

## Dependency Scanning (SCA)

Software Composition Analysis identifies known vulnerabilities in third-party libraries. Most applications are 80–90% third-party code.

### Common SCA Tools

| Tool | Ecosystem | Open Source | Notes |
|------|-----------|-------------|-------|
| **Trivy** | All (containers, IaC, code) | ✅ | Multi-purpose, fast |
| **Snyk** | npm, pip, Maven, Go, etc. | ❌ (free tier) | Developer-friendly, fix PRs |
| **Dependabot** | GitHub-native | ✅ | Automatic update PRs |
| **Renovate** | GitLab, GitHub, Bitbucket | ✅ | Highly configurable |
| **OWASP Dependency-Check** | Java, .NET | ✅ | NVD-based |
| **npm audit** / `pip audit` | npm / pip | ✅ | Built-in to package managers |

### Running Scans

```bash
# Trivy — scan a project directory
trivy fs --severity HIGH,CRITICAL --exit-code 1 .

# Trivy — scan a container image
trivy image --severity HIGH,CRITICAL myapp:latest

# npm audit
npm audit --audit-level=high

# pip-audit
pip-audit --strict --desc

# Go
govulncheck ./...
```

### CI Integration

```yaml
dependency-scan:
  stage: test
  image: aquasec/trivy
  script:
    - trivy fs --severity HIGH,CRITICAL --exit-code 1 --format json --output trivy-results.json .
  artifacts:
    reports:
      dependency_scanning: trivy-results.json
  allow_failure: false
```

---

## Secrets Detection

Detect hardcoded secrets (API keys, passwords, tokens) before they reach the repository.

### Tools

| Tool | Stage | Open Source |
|------|-------|-------------|
| **gitleaks** | Pre-commit, CI | ✅ |
| **trufflehog** | CI, post-commit scanning | ✅ |
| **detect-secrets** (Yelp) | Pre-commit | ✅ |

### Pre-Commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
```

```bash
# Manual scan
gitleaks detect --source . --verbose
```

---

## Penetration Testing Basics

Penetration testing is structured, authorised testing that simulates real-world attacks.

### Methodology (Simplified)

| Phase | Activities |
|-------|-----------|
| **1. Reconnaissance** | Discover endpoints, map attack surface, identify technologies |
| **2. Enumeration** | Find parameters, hidden endpoints, version information |
| **3. Vulnerability identification** | Test for OWASP Top 10 issues against each endpoint |
| **4. Exploitation** | Prove impact (data access, privilege escalation) |
| **5. Reporting** | Document findings with severity, evidence, and remediation |

### Manual Testing Checklist

| Category | Test |
|----------|------|
| **Authentication** | Default credentials, brute force, session fixation, MFA bypass |
| **Authorisation** | IDOR (change IDs), horizontal/vertical privilege escalation |
| **Injection** | SQLi, XSS, command injection in all input points |
| **Business logic** | Price manipulation, race conditions, workflow bypass |
| **Information disclosure** | Verbose errors, stack traces, debug endpoints, `.env` exposure |
| **Headers** | Missing security headers, misconfigured CORS |

### Useful Testing Tools

```bash
# Discover hidden endpoints and files
ffuf -u https://target.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt

# Test for SQL injection (authorised testing only)
sqlmap -u "https://target.com/api/users?id=1" --batch --level=3

# Test HTTP methods
curl -X OPTIONS https://target.com/api/users -i

# Check for open redirects
curl -v "https://target.com/login?redirect=https://evil.com"
```

---

## Security Headers Audit

Automated tools to verify your production headers:

```bash
# securityheaders.com — online scanner
# https://securityheaders.com/?q=yourdomain.com

# Mozilla Observatory — comprehensive grading
# https://observatory.mozilla.org/

# Command-line check
curl -sI https://yourdomain.com | grep -iE '^(strict-transport|content-security|x-content-type|x-frame|referrer-policy|permissions-policy)'
```

### Expected Output

```text
strict-transport-security: max-age=63072000; includeSubDomains; preload
content-security-policy: default-src 'self'; script-src 'self'; frame-ancestors 'none'
x-content-type-options: nosniff
x-frame-options: DENY
referrer-policy: strict-origin-when-cross-origin
permissions-policy: camera=(), microphone=(), geolocation=()
```

If any of these are missing, your application has a security gap.

---

## Incident Response

When a security incident occurs, the speed and quality of response determines the impact.

### Incident Response Phases

```text
┌────────────┐    ┌────────────────┐    ┌─────────────┐    ┌───────────┐
│ Detection   │──▶│ Containment    │──▶│ Eradication  │──▶│ Recovery  │
│             │    │                │    │              │    │           │
│ Alert fires │    │ Isolate system │    │ Fix root     │    │ Restore   │
│ User report │    │ Revoke tokens  │    │ cause        │    │ service   │
│ Log anomaly │    │ Block attacker │    │ Patch vuln   │    │ Verify    │
└────────────┘    └────────────────┘    └─────────────┘    └─────┬─────┘
                                                                  │
                                                                  ▼
                                                          ┌──────────────┐
                                                          │ Post-Mortem   │
                                                          │               │
                                                          │ Timeline      │
                                                          │ Root cause    │
                                                          │ Action items  │
                                                          └──────────────┘
```

### Immediate Actions for Common Incidents

| Incident | Immediate Response |
|----------|-------------------|
| **Credential leak** (token/key in git) | Rotate the credential immediately, scan for unauthorised usage |
| **SQL injection confirmed** | Take endpoint offline or deploy WAF rule, investigate data access |
| **Session hijacking** | Invalidate all sessions, force re-authentication |
| **Data breach** | Preserve evidence (logs), notify legal/compliance, contain the scope |
| **Dependency CVE** | Assess exploitability, patch or mitigate, deploy update |

### Post-Mortem Template

```markdown
## Incident Post-Mortem: [Title]

**Date:** YYYY-MM-DD
**Severity:** SEV-1 / SEV-2 / SEV-3
**Duration:** HH:MM

### Summary
One-paragraph description of what happened.

### Timeline
- HH:MM — Alert triggered
- HH:MM — Investigation started
- HH:MM — Root cause identified
- HH:MM — Fix deployed
- HH:MM — Incident resolved

### Root Cause
Technical explanation of what went wrong.

### Impact
- Users affected: N
- Data exposed: describe scope
- Financial impact: if applicable

### Action Items
| # | Action | Owner | Due Date |
|---|--------|-------|----------|
| 1 | Fix the vulnerability | @engineer | YYYY-MM-DD |
| 2 | Add regression test | @engineer | YYYY-MM-DD |
| 3 | Improve alerting | @ops | YYYY-MM-DD |
```

---

## Key Takeaways

- **Shift security left** — integrate SAST (Semgrep, CodeQL) and dependency scanning (Trivy, npm audit) into CI pipelines so vulnerabilities are caught before merge.
- **DAST complements SAST** — static analysis finds code-level flaws; dynamic analysis finds runtime misconfigurations, authentication issues, and business logic errors.
- **Dependency scanning** is non-negotiable — most applications are 80–90% third-party code, and known CVEs in dependencies are the easiest attack vector.
- **Secrets detection** (gitleaks, trufflehog) in pre-commit hooks and CI prevents the most common and most embarrassing security failures.
- **Security headers auditing** is a five-minute check that catches low-hanging fruit — use securityheaders.com and Mozilla Observatory regularly.
- **Incident response readiness** (documented playbooks, post-mortems, credential rotation procedures) determines whether a security event is a minor incident or a catastrophic breach.
