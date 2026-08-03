---
title: "Infrastructure Automation"
weight: 7
---

# Infrastructure Automation

Infrastructure automation brings the same rigour to infrastructure that CI/CD brings to application code: version control, peer review, automated testing, and repeatable deployments. GitOps, Terraform in pipelines, and policy-as-code are the pillars.

---

## GitOps

GitOps uses Git as the single source of truth for infrastructure and application state. A controller running in the cluster continuously reconciles the live state against what's declared in Git.

### Core Principles

| Principle | Meaning |
|-----------|---------|
| Declarative | Desired state is described, not scripted |
| Versioned and immutable | Git history is the audit trail |
| Pulled automatically | Controller pulls from Git, not pushed by CI |
| Continuously reconciled | Drift is detected and corrected automatically |

### Push vs Pull Deployment

```text
Push (traditional CI/CD):
  CI Pipeline ──push──▶ kubectl apply / helm upgrade ──▶ Cluster

Pull (GitOps):
  Developer ──commit──▶ Git Repo ◀──poll/watch── Controller ──reconcile──▶ Cluster
```

### ArgoCD

The most popular GitOps controller for Kubernetes:

```yaml
# ArgoCD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: apps/my-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true    # revert manual changes
    syncOptions:
      - CreateNamespace=true
```

Key features:
- Web UI with real-time sync status visualisation
- Multi-cluster support
- SSO integration (OIDC, LDAP, SAML)
- Rollback to any previous Git commit
- Health checks and custom health assessments

### Flux

A lightweight, composable alternative:

```yaml
# Flux GitRepository source
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/k8s-manifests.git
  ref:
    branch: main
---
# Flux Kustomization (reconciler)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/my-app/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-app
```

### ArgoCD vs Flux

| Aspect | ArgoCD | Flux |
|--------|--------|------|
| Architecture | Single controller + UI server | Set of small controllers |
| UI | Rich web dashboard | CLI-first (optional Weave UI) |
| Multi-tenancy | AppProjects | Namespaced GitRepositories |
| Helm support | Native | Via HelmRelease CRD |
| Learning curve | Moderate (UI helps) | Lower (YAML-native) |
| Best for | Teams wanting visibility + UI | Teams preferring composability |

---

## IaC in Pipelines

### Terraform in CI/CD

A standard pipeline for Terraform:

```yaml
# GitLab CI
stages:
  - validate
  - plan
  - apply

validate:
  stage: validate
  script:
    - terraform init -backend=false
    - terraform fmt -check
    - terraform validate

plan:
  stage: plan
  script:
    - terraform init -backend-config=config/backend/${ENV}.tfbackend
    - terraform plan -var-file=config/variables/${ENV}.tfvars -out=plan.tfplan
  artifacts:
    paths:
      - plan.tfplan

apply:
  stage: apply
  script:
    - terraform init -backend-config=config/backend/${ENV}.tfbackend
    - terraform apply plan.tfplan
  when: manual   # require human approval for production
  only:
    - main
```

### Pipeline Safety Rules

| Rule | Reason |
|------|--------|
| Always `plan` before `apply` | Catch unintended changes |
| Use saved plan files (`-out=plan.tfplan`) | Apply exactly what was reviewed |
| Lock state during apply | Prevent concurrent modifications |
| Require manual approval for prod | Human gate for destructive changes |
| Pin provider and module versions | Reproducible builds |
| Never store secrets in tfvars | Use secret managers or CI variables |

---

## Environment Promotion

Move infrastructure changes through environments with increasing confidence:

```text
┌──────────┐     ┌──────────┐     ┌──────────┐
│   Test   │────▶│   Pre    │────▶│   Prod   │
│ (auto)   │     │ (auto)   │     │ (manual) │
└──────────┘     └──────────┘     └──────────┘
     │                │                │
     ▼                ▼                ▼
  Smoke tests    Integration       Canary/gradual
  + plan check   tests + soak      + monitoring
```

### Promotion Strategies

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| Branch-per-env | `env/test`, `env/prod` branches | Simple, small teams |
| Directory-per-env | `environments/test/`, `environments/prod/` | Monorepo IaC |
| Parameterised (tfvars) | Same code, different variable files | Terraform projects |
| Promotion pipeline | Same artifact flows through stages | GitOps + ArgoCD |

---

## Drift Detection

Drift occurs when live infrastructure diverges from what's declared in code (manual console changes, out-of-band scripts).

### Detection Methods

```bash
# Terraform: compare state to reality
terraform plan -detailed-exitcode
# Exit code 2 = drift detected

# Schedule periodic drift checks in CI
drift-check:
  script:
    - terraform plan -detailed-exitcode -var-file=config/variables/prod.tfvars
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
```

### Drift Response Strategies

| Strategy | Action | When to Use |
|----------|--------|-------------|
| Alert only | Notify team, don't auto-fix | Production resources with manual exceptions |
| Auto-remediate | Re-apply desired state | Non-critical resources, GitOps clusters |
| Import + update code | Bring manual change into IaC | Legitimate out-of-band change |
| Ignore (lifecycle) | `ignore_changes` in Terraform | Auto-scaling counts, external tags |

---

## Policy as Code

Enforce governance rules automatically, before resources are deployed.

### Open Policy Agent (OPA) / Rego

General-purpose policy engine:

```rego
# policy/terraform.rego
package terraform

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  not resource.change.after.server_side_encryption_configuration
  msg := sprintf("S3 bucket '%s' must have encryption enabled", [resource.address])
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group_rule"
  resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
  resource.change.after.type == "ingress"
  msg := sprintf("Security group '%s' must not allow 0.0.0.0/0 ingress", [resource.address])
}
```

Integrate with Terraform:

```bash
# Generate plan JSON
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json

# Evaluate policies
opa eval --data policy/ --input plan.json "data.terraform.deny"
```

### HashiCorp Sentinel

Policy-as-code for Terraform Cloud/Enterprise:

```python
# Sentinel policy
import "tfplan/v2" as tfplan

s3_buckets = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket" and rc.mode is "managed"
}

main = rule {
  all s3_buckets as _, bucket {
    bucket.change.after.server_side_encryption_configuration is not null
  }
}
```

### Comparison

| Tool | Integration | Language | Best For |
|------|-------------|----------|----------|
| OPA / Conftest | Any CI, Kubernetes (Gatekeeper) | Rego | Multi-platform, K8s admission |
| Sentinel | Terraform Cloud/Enterprise only | Sentinel (Python-like) | Terraform-native governance |
| Checkov | CLI, CI plugins | Python (YAML rules) | IaC scanning (Terraform, CloudFormation, K8s) |
| tfsec / Trivy | CLI, CI plugins | Go (built-in rules) | Quick Terraform security checks |

### Pipeline Integration

```yaml
policy-check:
  stage: validate
  script:
    - terraform plan -out=plan.tfplan
    - terraform show -json plan.tfplan > plan.json
    - conftest test plan.json --policy policy/
  allow_failure: false
```

---

## Repository Patterns

### App Repo vs Config Repo (GitOps)

| Pattern | Structure | Pros | Cons |
|---------|-----------|------|------|
| Monorepo | App code + K8s manifests together | Simple, atomic changes | Image build triggers manifest redeploy |
| Split repos | App repo + separate config/infra repo | Clean separation, independent lifecycles | Coordination overhead |

Recommended split for GitOps:
- **App repo** → builds image, pushes to registry, updates image tag in config repo
- **Config repo** → K8s manifests / Helm values, watched by ArgoCD/Flux

---

## Key Takeaways

1. **GitOps** makes Git the source of truth for cluster state — changes are auditable, reversible, and reconciled automatically.
2. **ArgoCD** suits teams wanting a UI and multi-cluster visibility; **Flux** suits teams preferring lightweight composability.
3. **Terraform in CI/CD** must always plan before apply, use saved plan files, and gate production with manual approval.
4. **Environment promotion** should flow test → pre → prod with increasing gate strictness.
5. **Drift detection** on a schedule catches console cowboys — decide whether to alert or auto-remediate based on the resource's criticality.
6. **Policy as code** (OPA, Sentinel, Checkov) prevents misconfigurations from ever reaching infrastructure — enforce encryption, restrict public access, and mandate tagging automatically.
