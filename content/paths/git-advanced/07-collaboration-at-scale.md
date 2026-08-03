---
title: "Collaboration at Scale"
weight: 7
---

# Collaboration at Scale

When teams grow beyond a handful of developers, Git workflows become critical infrastructure. The right workflow reduces merge conflicts, accelerates delivery, and preserves code quality. The wrong one creates bottlenecks and frustration.

---

## Git Workflows Compared

### GitFlow

```
main ─────────●─────────────────●──────────── (releases)
               ╲               ╱
develop ────●───●───●───●───●─●──────────── (integration)
             ╲     ╱   ╲   ╱
feature/x ────●───●     ╲ ╱
                     feature/y ──●
```

| Aspect | Details |
|--------|---------|
| **Branches** | main, develop, feature/*, release/*, hotfix/* |
| **Merge style** | Merge commits (no fast-forward) |
| **Release cycle** | Planned releases with release branches |
| **Best for** | Packaged software, mobile apps, versioned products |
| **Drawback** | Complex, long-lived branches, integration pain |

### GitHub Flow

```
main ─────●──────●──────●──────●──────── (always deployable)
           ╲    ╱  ╲   ╱  ╲   ╱
feature ────●──●    ●──●    ●──●
            PR+CI   PR+CI   PR+CI
```

| Aspect | Details |
|--------|---------|
| **Branches** | main + short-lived feature branches |
| **Merge style** | Squash or merge commit via Pull Request |
| **Release cycle** | Deploy on every merge to main |
| **Best for** | SaaS, web applications, continuous deployment |
| **Drawback** | Requires robust CI/CD and feature flags |

### Trunk-Based Development

```
main ──●──●──●──●──●──●──●──●──●──── (everyone commits here)
        │  │     │        │
        small    small    small
        commits  commits  commits
```

| Aspect | Details |
|--------|---------|
| **Branches** | main only (or very short-lived branches < 1 day) |
| **Merge style** | Direct commits or same-day PRs |
| **Release cycle** | Continuous; feature flags gate incomplete work |
| **Best for** | High-performing teams, CI/CD maturity, Google/Meta-style |
| **Drawback** | Requires excellent test coverage and feature flags |

### Ship / Show / Ask

A nuanced model that categorizes changes by risk:

| Category | Process | Example |
|----------|---------|---------|
| **Ship** | Merge directly to main, no review | Typo fix, config change |
| **Show** | Merge to main, open PR for async review | Refactoring with tests |
| **Ask** | Open PR, wait for review before merge | New feature, architecture change |

| Aspect | Details |
|--------|---------|
| **Philosophy** | Trust engineers to categorize their own changes |
| **Best for** | Experienced teams with high trust and good test coverage |
| **Drawback** | Requires strong team culture and judgment |

### Comparison Table

| Criterion | GitFlow | GitHub Flow | Trunk-Based | Ship/Show/Ask |
|-----------|---------|-------------|-------------|---------------|
| Complexity | High | Low | Lowest | Low |
| Branch lifespan | Days-weeks | Hours-days | Minutes-hours | Varies |
| Merge conflicts | Frequent | Occasional | Rare | Rare |
| CI/CD requirement | Medium | High | Very High | High |
| Team size sweet spot | 5-20 | 3-50 | 5-100+ | 3-15 |
| Release cadence | Weekly-monthly | Daily | Continuous | Continuous |

---

## Large Repository Strategies

### Monorepo

All code in a single repository:

```
monorepo/
├── services/
│   ├── orders/
│   ├── payments/
│   └── inventory/
├── libraries/
│   ├── shared-auth/
│   └── common-types/
└── infrastructure/
    └── terraform/
```

| Pros | Cons |
|------|------|
| Atomic cross-service changes | Git performance degrades at scale |
| Shared tooling and CI | Everyone sees all changes (noise) |
| Easy code reuse | Requires build system sophistication |
| Single version of truth | Ownership boundaries blur |

**Tools:** Nx, Turborepo, Bazel, Pants, Rush

### Sparse Checkout (Git native)

```bash
# Only check out what your team needs
git sparse-checkout init --cone
git sparse-checkout set services/orders libraries/shared-auth
```

### Git Submodules / Subtrees

| Approach | Use Case | Complexity |
|----------|----------|------------|
| Submodules | Pin external dependency at specific commit | High (confusing for newcomers) |
| Subtrees | Embed external repo in your tree | Medium (merge-based) |

---

## Signed Commits

Verify that commits genuinely came from the claimed author:

```bash
# Setup GPG signing
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true

# Sign a commit
git commit -S -m "feat: add payment validation"

# Verify
git log --show-signature
# gpg: Signature made ... using RSA key ID ABC123
# gpg: Good signature from "Developer Name <dev@example.com>"
```

### SSH Signing (Git 2.34+)

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
```

### Enforcement

| Platform | Setting |
|----------|---------|
| GitHub | Branch protection → Require signed commits |
| GitLab | Push rules → Reject unsigned commits |
| Local | `git config --global commit.gpgsign true` |

---

## Merge Queues

Merge queues solve the "semantic conflict" problem: two PRs pass CI individually but break when combined.

### The Problem

```
PR #1 passes CI  ✓  (changes function signature)
PR #2 passes CI  ✓  (calls function with old signature)
Both merge → main is broken ✗
```

### How Merge Queues Work

```
PR #1 ──→ ┌─────────────┐
           │ Merge Queue │ → Test (main + PR#1 + PR#2) → Merge both
PR #2 ──→ └─────────────┘                              or reject PR#2
```

1. PR is approved and added to the queue
2. Queue creates a speculative merge (main + all queued PRs)
3. CI runs on the combined result
4. If it passes, all queued PRs merge; if not, the failing PR is ejected

| Platform | Feature |
|----------|---------|
| GitHub | Merge queue (built-in) |
| GitLab | Merge trains |
| Bors-ng | Third-party bot |

---

## Code Review Automation

### CODEOWNERS

Automatically assign reviewers based on file paths:

```bash
# .github/CODEOWNERS (GitHub) or CODEOWNERS (GitLab)

# Global default
*                       @team-leads

# Service-specific ownership
/services/orders/       @orders-team
/services/payments/     @payments-team @security-team

# Infrastructure requires platform team
/infrastructure/        @platform-team
*.tf                    @platform-team

# Documentation — anyone can review
/docs/                  @tech-writers

# Security-sensitive files need security review
/auth/                  @security-team
**/secrets*             @security-team
```

### Automated Checks

| Tool | Purpose |
|------|---------|
| **Danger.js / Danger.rb** | Custom PR rules (size limits, changelog, labels) |
| **Semantic PR** | Enforce conventional commit titles |
| **Reviewdog** | Post linter results as PR comments |
| **CodeRabbit / Copilot** | AI-powered code review suggestions |

### Review Policies

```yaml
# Example: GitLab merge request approval rules
approvals_required: 2
rules:
  - name: "Security Review"
    approvals_required: 1
    groups: ["security-team"]
    applies_to: ["auth/**", "*.tf"]
  - name: "Standard Review"
    approvals_required: 1
    groups: ["developers"]
```

### PR Size Guidelines

| Size (lines changed) | Category | Expected Review Time |
|---------------------|----------|---------------------|
| 1-50 | Small | < 30 minutes |
| 51-200 | Medium | 1-2 hours |
| 201-500 | Large | Half day (consider splitting) |
| 500+ | Too large | Must be split |

> **Rule of thumb:** If a PR takes more than 30 minutes to review, it's too large. Reviewers lose attention after ~400 lines.

---

## Scaling Practices Summary

| Challenge | Solution |
|-----------|----------|
| Merge conflicts | Short-lived branches, trunk-based development |
| Broken main | Merge queues, required CI checks |
| Unclear ownership | CODEOWNERS, team-based directories |
| Review bottlenecks | Auto-assignment, review SLAs, stack PRs |
| Commit authenticity | Signed commits (GPG/SSH) |
| Monorepo performance | Sparse checkout, build caching (Nx/Bazel) |
| Cross-team coordination | ADRs, architecture fitness functions |

---

## Key Takeaways

1. **Trunk-based development** scales best for high-performing teams — short-lived branches and feature flags replace complex branching models.
2. **GitHub Flow** is the pragmatic middle ground for most teams.
3. **Merge queues** prevent the "passing individually, failing together" problem.
4. **CODEOWNERS** ensures the right people review the right code automatically.
5. **Signed commits** prove authorship — essential for supply-chain security.
6. Keep PRs **small** (< 200 lines) — review quality drops dramatically with size.
7. The best workflow is the one your team **actually follows** — complexity kills adoption.
