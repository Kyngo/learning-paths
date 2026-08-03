---
title: "Artifact Management"
weight: 6
---

# Artifact Management

Artifacts — container images, packages, binaries — are the output of your CI pipeline and the input to your CD pipeline. Managing them well means consistent tagging, secure storage, vulnerability scanning, and retention policies.

---

## Container Registries

A container registry stores and distributes Docker/OCI images. Choose based on your ecosystem:

| Registry | Best For | Authentication | Key Features |
|----------|----------|---------------|--------------|
| Amazon ECR | AWS-native workloads | IAM roles | Lifecycle policies, image scanning, cross-region replication |
| Docker Hub | Open-source, public images | Username/token | Official images, automated builds, rate limits on free tier |
| GitLab Container Registry | GitLab CI users | CI_JOB_TOKEN | Built-in, zero config, per-project isolation |
| GitHub Container Registry (ghcr.io) | GitHub Actions users | GITHUB_TOKEN | Free for public images, org-level packages |
| Google Artifact Registry | GCP workloads | Service accounts | Multi-format (Docker, npm, Maven, Python) |
| Azure Container Registry | Azure/AKS users | Managed Identity | Geo-replication, ACR Tasks for in-registry builds |

### ECR Lifecycle Policy Example

Automatically clean old images to control storage costs:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 20 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 20
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Remove untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

---

## Artifact Repositories (Non-Container)

For JARs, npm packages, Python wheels, and generic binaries:

| Tool | Supported Formats | Deployment | Licence |
|------|------------------|------------|---------|
| JFrog Artifactory | All (Maven, npm, PyPI, Docker, Helm, Go, etc.) | Cloud or self-hosted | Commercial |
| Sonatype Nexus | Maven, npm, PyPI, Docker, NuGet, Helm | Self-hosted (OSS) or Pro | OSS + Commercial |
| GitHub Packages | npm, Maven, NuGet, Docker, RubyGems | Cloud (GitHub-integrated) | Free tier + paid |
| GitLab Package Registry | Maven, npm, PyPI, NuGet, Go, Composer | Built into GitLab | Free |
| AWS CodeArtifact | Maven, npm, PyPI, NuGet | AWS-managed | Pay-per-use |

### When to Use What

- **Small team, single language** → GitLab/GitHub built-in package registry
- **Multi-language monorepo or enterprise** → Artifactory or Nexus (central proxy + cache)
- **AWS-native, need upstream proxy** → CodeArtifact (proxies npmjs, PyPI, Maven Central)

---

## Semantic Versioning in Pipelines

Every artifact should carry a meaningful version. SemVer (`MAJOR.MINOR.PATCH`) is the standard:

| Segment | When to Increment | Example |
|---------|-------------------|---------|
| MAJOR | Breaking API/interface changes | 1.2.3 → 2.0.0 |
| MINOR | New features, backward-compatible | 1.2.3 → 1.3.0 |
| PATCH | Bug fixes, no API changes | 1.2.3 → 1.2.4 |

### Automating Version Bumps

Tools that derive the next version from commit messages (Conventional Commits):

```bash
# semantic-release (Node.js ecosystem)
npx semantic-release

# commitizen + standard-version
npx standard-version

# Python: python-semantic-release
semantic-release publish
```

Conventional Commits → version mapping:

| Commit Prefix | Version Bump |
|---------------|-------------|
| `fix:` | PATCH |
| `feat:` | MINOR |
| `feat!:` or `BREAKING CHANGE:` | MAJOR |
| `chore:`, `docs:`, `ci:` | No bump |

---

## Image Tagging Strategies

| Strategy | Format | Pros | Cons |
|----------|--------|------|------|
| Git SHA | `myapp:a1b2c3d` | Unique, traceable to commit | Not human-readable |
| SemVer tag | `myapp:1.4.2` | Clear versioning | Requires tagging discipline |
| Branch + SHA | `myapp:main-a1b2c3d` | Shows source branch | Cluttered |
| Latest | `myapp:latest` | Simple | Non-deterministic, breaks reproducibility |
| Timestamp | `myapp:20260803-141200` | Sortable | No semantic meaning |
| Combined | `myapp:1.4.2-a1b2c3d` | Version + traceability | Longer tag |

### Recommended Approach

```yaml
# GitLab CI example
variables:
  IMAGE_TAG: "${CI_COMMIT_TAG:-${CI_COMMIT_SHORT_SHA}}"
  IMAGE: "${CI_REGISTRY_IMAGE}:${IMAGE_TAG}"

build:
  script:
    - docker build -t $IMAGE .
    - docker push $IMAGE
    # Also tag as latest for convenience (non-prod only)
    - docker tag $IMAGE ${CI_REGISTRY_IMAGE}:latest
    - docker push ${CI_REGISTRY_IMAGE}:latest
```

### Immutable Tags

Production images should use **immutable tags** — once pushed, a tag cannot be overwritten:

```bash
# ECR: enable image tag immutability
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE
```

---

## Vulnerability Scanning in CI

Scan images and dependencies before they reach production:

### Scanning Tools

| Tool | Scans | Integration | Licence |
|------|-------|-------------|---------|
| Trivy | Container images, filesystem, IaC | CLI, CI plugins | OSS (Aqua) |
| Grype | Container images, SBOMs | CLI, GitHub Actions | OSS (Anchore) |
| Snyk | Containers, code, IaC, deps | CI/CD, IDE plugins | Freemium |
| Docker Scout | Docker images | Docker CLI, Hub | Built into Docker |
| ECR Scanning | ECR images (basic or enhanced) | AWS-native | Included with ECR |

### Trivy in CI Pipeline

```yaml
# GitLab CI
scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE
    - trivy image --format json --output trivy-report.json $IMAGE
  artifacts:
    paths:
      - trivy-report.json
  allow_failure: false
```

```yaml
# GitHub Actions
- name: Trivy vulnerability scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'HIGH,CRITICAL'
    exit-code: '1'

- name: Upload scan results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

### Gate Policy

| Severity | Action |
|----------|--------|
| Critical | Block deployment, require immediate fix |
| High | Block deployment to production |
| Medium | Warn, create ticket, allow deployment |
| Low | Report only |

### Shifting Left: Dependency Scanning

Scan dependencies before building the image:

```bash
# Python
pip-audit --requirement requirements.txt

# Node.js
npm audit --audit-level=high

# Go
govulncheck ./...
```

---

## Retention & Cleanup Policies

Unmanaged registries grow indefinitely. Define policies:

| Rule | Example |
|------|---------|
| Keep last N tagged images | 20 per repository |
| Delete untagged after N days | 7 days |
| Keep all release tags (vX.Y.Z) | Indefinitely |
| Delete feature-branch images after merge | Same day |
| Archive images older than 90 days | Move to cold storage or delete |

---

## Key Takeaways

1. **Use your platform's native registry** for simplicity (ECR for AWS, GitLab Registry for GitLab CI) — add Artifactory/Nexus only when you need cross-platform proxying.
2. **Tag images with SemVer + SHA** for both human readability and commit traceability. Never deploy `latest` to production.
3. **Enable immutable tags** in production registries to prevent accidental overwrites.
4. **Scan every image in CI** — fail the pipeline on Critical/High findings. Use Trivy or Grype as a free baseline.
5. **Automate cleanup** — lifecycle policies prevent registry bloat and reduce storage costs.
6. **Automate version bumps** from Conventional Commits — removes human error from the versioning process.
