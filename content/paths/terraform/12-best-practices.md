---
title: "Best Practices"
weight: 12
---

## Code Organization

### File Structure

```text
project/
├── main.tf              # Primary resources or module calls
├── variables.tf         # All input variables
├── outputs.tf           # All outputs
├── locals.tf            # Local values
├── data.tf              # Data sources
├── providers.tf         # Provider configuration
├── versions.tf          # Required providers and Terraform version
├── backend.tf           # Backend configuration
└── README.md            # Documentation
```

For larger projects, group by service:

```text
states/main/
├── backend.tf
├── provider.tf
├── variables.tf
├── locals.tf
├── data.tf
├── outputs.tf
└── services/
    ├── api/
    │   ├── ecs.tf
    │   ├── alb.tf
    │   └── outputs.tf
    └── worker/
        ├── ecs.tf
        ├── sqs.tf
        └── outputs.tf
```

### Naming Conventions

```hcl
# Resources: snake_case, descriptive
resource "aws_security_group" "api_server" { }
resource "aws_iam_role" "ecs_task_execution" { }
resource "aws_s3_bucket" "application_logs" { }

# Variables: snake_case, include unit if ambiguous
variable "cache_ttl_seconds" { }
variable "max_instance_count" { }
variable "volume_size_gb" { }

# Outputs: include resource context
output "vpc_id" { }
output "api_load_balancer_dns" { }
output "database_endpoint" { }

# Locals: snake_case, descriptive
locals {
  name_prefix   = "${var.project}-${var.environment}"
  is_production = var.environment == "prod"
}
```

### Single Resource Naming

When a module has only one of a resource type, use `this` or `main`:

```hcl
# In a VPC module — there's only one VPC
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
}

# In root — there's only one cluster
resource "aws_ecs_cluster" "main" {
  name = local.cluster_name
}
```

---

## Version Pinning

### Terraform Version

```hcl
terraform {
  required_version = "~> 1.11.0"  # allows 1.11.x, blocks 1.12+
}
```

### Provider Versions

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # allows 5.x, blocks 6.0
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

### Module Versions

```hcl
# Registry modules
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# Git modules — always pin to a tag
module "service" {
  source = "git::ssh://git@github.com/org/modules.git//ecs-service?ref=v3.2.1"
}
```

### Lock File

```bash
# .terraform.lock.hcl — commit this file!
# Records exact provider versions and hashes
# Ensures everyone uses identical provider binaries
terraform init -upgrade  # update lock file when changing versions
```

---

## Security

### Secrets Management

```hcl
# ❌ Never hardcode secrets
resource "aws_db_instance" "main" {
  password = "super-secret-123"  # NEVER DO THIS
}

# ✅ Use variables (set via CI/CD secrets or env vars)
resource "aws_db_instance" "main" {
  password = var.db_password
}

# ✅ Generate random passwords
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_db_instance" "main" {
  password = random_password.db.result
}

# ✅ Reference Secrets Manager
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/database/password"
}
```

### State Security

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-central-1"
    encrypt        = true                    # SSE-KMS encryption
    dynamodb_table = "terraform-locks"       # state locking
  }
}
```

Bucket policy:

- Deny unencrypted uploads
- Restrict access to CI/CD role and admins only
- Enable versioning (recover from corruption)
- Enable access logging

### .gitignore

```gitignore
# State files (contain secrets)
*.tfstate
*.tfstate.*

# Crash logs
crash.log
crash.*.log

# Variable files with secrets
*.tfvars
!example.tfvars

# Provider plugins (large, downloaded on init)
.terraform/

# Plan files (may contain secrets)
*.tfplan
```

---

## DRY Principles

### Use Locals for Repeated Expressions

```hcl
# ❌ Repeated everywhere
resource "aws_instance" "web" {
  tags = {
    Environment = var.environment
    Project     = var.project
    Team        = var.team
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket" "logs" {
  tags = {
    Environment = var.environment
    Project     = var.project
    Team        = var.team
    ManagedBy   = "terraform"
  }
}

# ✅ Define once, use everywhere
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project
    Team        = var.team
    ManagedBy   = "terraform"
  }
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, { Name = "${local.prefix}-web" })
}

resource "aws_s3_bucket" "logs" {
  tags = merge(local.common_tags, { Name = "${local.prefix}-logs" })
}
```

### Use for_each Over count

```hcl
# ❌ count — fragile when items are removed
resource "aws_iam_user" "team" {
  count = length(var.team_members)
  name  = var.team_members[count.index]
}

# ✅ for_each — stable, items have identity
resource "aws_iam_user" "team" {
  for_each = toset(var.team_members)
  name     = each.value
}
```

### Use Modules for Repeated Patterns

```hcl
# ❌ Copy-paste for each service
resource "aws_ecs_service" "api" { /* 50 lines */ }
resource "aws_ecs_task_definition" "api" { /* 40 lines */ }
resource "aws_cloudwatch_log_group" "api" { /* 10 lines */ }

resource "aws_ecs_service" "worker" { /* same 50 lines */ }
resource "aws_ecs_task_definition" "worker" { /* same 40 lines */ }
resource "aws_cloudwatch_log_group" "worker" { /* same 10 lines */ }

# ✅ Module encapsulates the pattern
module "api_service" {
  source = "./modules/ecs-service"
  name   = "api"
  image  = var.api_image
  port   = 8080
}

module "worker_service" {
  source = "./modules/ecs-service"
  name   = "worker"
  image  = var.worker_image
  port   = 0
}
```

---

## Operational Practices

### Run Plan Regularly

```bash
# Detect drift even without changes
terraform plan
# If "No changes" — infrastructure matches config
# If changes shown — someone modified infrastructure manually
```

### Use Terraform fmt

```bash
# Format all files
terraform fmt -recursive

# Check formatting (CI/CD)
terraform fmt -check -recursive
```

### Document with terraform-docs

```bash
# Auto-generate documentation from variables/outputs
terraform-docs markdown table . > TERRAFORM.md
```

### Limit Blast Radius

- Separate state per environment
- Separate state per component (networking, compute, database)
- Use `-target` only for debugging, never in normal workflow
- Small, frequent applies over large, infrequent ones

---

## Common Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Monolithic state | Slow plans, large blast radius | Split by component |
| Hardcoded values | Not reusable | Variables + tfvars |
| No version pinning | Unexpected breakage | Pin everything |
| Manual applies | No audit trail, human error | CI/CD pipeline |
| Secrets in code | Security breach | Secrets Manager, env vars |
| `count` for named things | Index shifting on removal | `for_each` |
| Deep module nesting | Hard to understand | Flat composition |
| Ignoring plan output | Unexpected destroys | Always review plans |

---

## Key Takeaways

1. **Pin all versions** — Terraform, providers, and modules
2. **Commit the lock file** — ensures reproducible builds
3. **Never commit secrets** — use Secrets Manager, env vars, or Vault
4. **Encrypt and lock state** — it contains your infrastructure's secrets
5. **Use `for_each` over `count`** — stable identifiers prevent accidental destroys
6. **Modules for repeated patterns** — DRY without sacrificing clarity
7. **Small state, small blast radius** — separate concerns into independent states
8. **Format and validate in CI** — catch issues before they reach infrastructure
9. **Review every plan** — the plan is your safety net; never skip it
