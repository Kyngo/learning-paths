---
title: "Testing & Validation"
weight: 13
---

# Testing & Validation

Terraform code manages production infrastructure. Bugs don't just break builds — they delete databases, expose services to the internet, or incur unexpected costs. Testing Terraform is not optional.

---

## The Terraform Testing Pyramid

```
      ╱╲
     ╱  ╲  E2E (apply to real cloud, verify)
    ╱────╲
   ╱      ╲  Integration (plan against real providers)
  ╱────────╲
 ╱          ╲  Static analysis (lint, validate, scan)
╱────────────╲
```

| Level | Speed | Cost | Catches |
|-------|-------|------|---------|
| Static analysis | Seconds | Free | Syntax errors, policy violations, security misconfigurations |
| Integration (plan) | Minutes | Minimal (API calls for plan, no creates) | Configuration logic errors, dependency issues |
| E2E (apply + verify) | Minutes-hours | Cloud resource costs | Real infrastructure behaviour |

---

## Level 1: Static Analysis

### `terraform validate`

Checks syntax and internal consistency (no cloud API calls):

```bash
terraform validate
# Success! The configuration is valid.
```

Catches: missing required arguments, type mismatches, invalid references, unknown resource attributes.

### `terraform fmt`

Enforce canonical formatting:

```bash
terraform fmt -check -recursive    # fail if unformatted (CI)
terraform fmt -recursive           # fix all files
```

### TFLint

Linter that catches issues `validate` misses:

```bash
tflint --init
tflint --recursive
```

| Rule | Example |
|------|---------|
| Invalid instance type | `aws_instance` with `"t2.superlarge"` |
| Deprecated resources | Using `aws_iam_policy_attachment` instead of `aws_iam_role_policy_attachment` |
| Naming conventions | Non-snake_case resource names |
| Missing tags | Resources without required tags |

### Configuration (`.tflint.hcl`)

```hcl
plugin "aws" {
  enabled = true
  version = "0.30.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}
```

---

## Level 2: Security Scanning

### Checkov

Scans for security misconfigurations against 1000+ built-in policies:

```bash
checkov -d .
checkov -f main.tf
checkov --framework terraform --compact
```

| Finding | Policy |
|---------|--------|
| S3 bucket without encryption | CKV_AWS_19 |
| Security group with 0.0.0.0/0 on port 22 | CKV_AWS_24 |
| RDS without encryption at rest | CKV_AWS_16 |
| IAM policy with * resource | CKV_AWS_62 |
| CloudTrail not enabled | CKV_AWS_35 |

### tfsec (Now Part of Trivy)

```bash
trivy config .
```

Similar coverage to Checkov, integrated with the Trivy vulnerability scanner.

### KICS (Keeping Infrastructure as Code Secure)

```bash
kics scan -p .
```

Broader coverage (Terraform, CloudFormation, Kubernetes, Docker, Ansible).

---

## Level 3: Policy as Code

### Open Policy Agent (OPA) / Conftest

Write custom policies in Rego:

```rego
# policy/terraform.rego
package main

deny[msg] {
    resource := input.resource.aws_s3_bucket[name]
    not resource.server_side_encryption_configuration
    msg := sprintf("S3 bucket '%s' must have encryption enabled", [name])
}

deny[msg] {
    resource := input.resource.aws_instance[name]
    resource.instance_type == "m5.4xlarge"
    msg := sprintf("Instance '%s' uses expensive type m5.4xlarge — use m5.xlarge or smaller", [name])
}
```

```bash
terraform show -json tfplan > plan.json
conftest test plan.json --policy policy/
```

### HashiCorp Sentinel (Enterprise)

Policy-as-code framework built into Terraform Cloud/Enterprise:

```python
import "tfplan/v2" as tfplan

main = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is not "aws_iam_user_policy_attachment"
    }
}
```

---

## Level 4: Integration Testing (Plan)

### Terraform Test (Native, 1.6+)

```hcl
# tests/vpc.tftest.hcl
run "verify_vpc" {
  command = plan  # or apply

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block must be 10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames must be enabled"
  }
}

run "verify_subnets" {
  command = plan

  assert {
    condition     = length(aws_subnet.private) == 3
    error_message = "Expected 3 private subnets"
  }
}
```

```bash
terraform test
```

### Terratest (Go-Based E2E)

For full apply-verify-destroy cycles:

```go
func TestVpc(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "cidr_block": "10.0.0.0/16",
            "name":       "test-vpc",
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)

    vpc := aws.GetVpcById(t, vpcId, "eu-west-1")
    assert.Equal(t, "10.0.0.0/16", vpc.CidrBlock)
}
```

---

## CI Pipeline Pattern

```yaml
stages:
  - validate
  - plan
  - apply

validate:
  script:
    - terraform fmt -check -recursive
    - terraform validate
    - tflint --recursive
    - checkov -d . --compact
    - terraform test

plan:
  script:
    - terraform plan -out=tfplan
    - terraform show -json tfplan > plan.json
    - conftest test plan.json --policy policy/
  artifacts:
    paths: [tfplan]

apply:
  script:
    - terraform apply tfplan
  when: manual  # require human approval
  only:
    - main
```

---

## Key Takeaways

- `terraform validate` + `terraform fmt` are the bare minimum. Run them in every CI pipeline.
- TFLint catches cloud-specific issues that `validate` misses (invalid instance types, deprecated resources).
- Checkov/Trivy scan for security misconfigurations against 1000+ policies. Run before every plan.
- OPA/Conftest enable custom organisational policies (cost guardrails, naming conventions, tag requirements).
- `terraform test` (native since 1.6) handles plan-level assertions. Terratest handles full apply-destroy cycles.
- The CI pipeline should be: format → validate → lint → security scan → plan → policy check → manual apply.
