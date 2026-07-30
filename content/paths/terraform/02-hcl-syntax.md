---
title: "HCL Syntax"
weight: 2
---

## What is HCL?

HCL (HashiCorp Configuration Language) is a domain-specific language designed for infrastructure configuration. It's not a general-purpose programming language — it's purpose-built to be human-readable, machine-parseable, and expressive enough to describe complex infrastructure.

---

## Block Structure

Everything in HCL is organized into blocks:

```hcl
block_type "label_1" "label_2" {
  argument = value
  
  nested_block {
    nested_argument = value
  }
}
```

### Common Block Types

| Block | Labels | Purpose |
|-------|--------|---------|
| `resource` | type, name | Create infrastructure |
| `data` | type, name | Read existing infrastructure |
| `variable` | name | Declare input |
| `output` | name | Expose values |
| `locals` | (none) | Define computed values |
| `module` | name | Use a reusable module |
| `provider` | name | Configure a provider |
| `terraform` | (none) | Terraform settings |

```hcl
# Resource block: 2 labels (type + local name)
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-data-lake"
}

# Variable block: 1 label (name)
variable "region" {
  type    = string
  default = "eu-central-1"
}

# Locals block: no labels
locals {
  prefix = "myapp-${var.environment}"
}
```

---

## Types and Values

### Primitive Types

```hcl
# String
name = "web-server"

# Number (integer or float)
port    = 8080
timeout = 3.5

# Boolean
enabled = true
```

### Collection Types

```hcl
# List (ordered, duplicates allowed)
availability_zones = ["eu-central-1a", "eu-central-1b", "eu-central-1c"]

# Map (key-value pairs)
tags = {
  Environment = "prod"
  Team        = "platform"
}

# Set (unordered, no duplicates)
allowed_ips = toset(["10.0.0.1", "10.0.0.2"])
```

### Structural Types

```hcl
# Object (fixed structure with named attributes)
variable "database" {
  type = object({
    engine         = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
}

# Tuple (fixed-length, mixed types)
variable "rule" {
  type = tuple([string, number, bool])
  # Example: ["allow", 443, true]
}
```

### Type Constraints

```hcl
variable "ports" {
  type = list(number)  # list of numbers only
}

variable "tags" {
  type = map(string)  # map with string values only
}

variable "services" {
  type = list(object({
    name = string
    port = number
    path = optional(string, "/")  # optional with default
  }))
}
```

---

## Expressions

### String Interpolation and Templates

```hcl
# Simple interpolation
name = "app-${var.environment}-${var.region}"

# Heredoc (multi-line strings)
policy = <<-EOT
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${var.bucket_name}/*"
    }]
  }
EOT

# Directive (template logic)
user_data = <<-EOT
  #!/bin/bash
  %{ for port in var.ports ~}
  ufw allow ${port}
  %{ endfor ~}
EOT
```

### Operators

```hcl
# Arithmetic
total = var.base_count * var.multiplier

# Comparison
is_large = var.instance_count > 10

# Logical
needs_scaling = var.is_production && var.high_traffic
```

### Conditional Expressions

```hcl
# condition ? true_value : false_value
instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

# Nested (avoid — use locals for readability)
size = var.tier == "large" ? 100 : var.tier == "medium" ? 50 : 20
```

### For Expressions

```hcl
# Transform a list
upper_names = [for name in var.names : upper(name)]

# Filter a list
prod_instances = [for i in var.instances : i if i.environment == "prod"]

# List to map
instance_map = { for i in var.instances : i.name => i.id }

# Map transformation
uppercased = { for k, v in var.tags : k => upper(v) }
```

### Splat Expressions

```hcl
# Shorthand for [for o in list : o.attribute]
instance_ids = aws_instance.web[*].id
subnet_cidrs = aws_subnet.private[*].cidr_block

# Equivalent to:
instance_ids = [for i in aws_instance.web : i.id]
```

---

## References

### Resource References

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id          # reference another resource
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, 0)
}
```

### Reference Types

| Reference | Syntax | Example |
|-----------|--------|---------|
| Resource attribute | `type.name.attribute` | `aws_vpc.main.id` |
| Variable | `var.name` | `var.environment` |
| Local | `local.name` | `local.common_tags` |
| Module output | `module.name.output` | `module.vpc.vpc_id` |
| Data source | `data.type.name.attribute` | `data.aws_ami.latest.id` |
| Count index | `count.index` | `count.index` |
| Each key/value | `each.key`, `each.value` | `each.value.port` |
| Self (provisioners) | `self.attribute` | `self.public_ip` |

---

## Dynamic Blocks

Generate repeated nested blocks from a collection:

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

Use dynamic blocks when the number of nested blocks varies. Avoid them when the structure is fixed — explicit blocks are more readable.

---

## Built-in Functions

HCL has no user-defined functions, but provides a rich standard library:

### String Functions

```hcl
upper("hello")                    # "HELLO"
lower("Hello")                    # "hello"
replace("hello world", " ", "-")  # "hello-world"
split(",", "a,b,c")              # ["a", "b", "c"]
join("-", ["a", "b", "c"])       # "a-b-c"
format("Hello, %s!", var.name)   # "Hello, World!"
trimspace("  hello  ")           # "hello"
```

### Numeric Functions

```hcl
min(1, 2, 3)     # 1
max(1, 2, 3)     # 3
ceil(4.2)        # 5
floor(4.8)       # 4
abs(-5)          # 5
```

### Collection Functions

```hcl
length(["a", "b", "c"])                    # 3
contains(["a", "b"], "a")                  # true
merge({a = 1}, {b = 2})                    # {a = 1, b = 2}
lookup({a = 1, b = 2}, "a", 0)            # 1
flatten([["a", "b"], ["c"]])              # ["a", "b", "c"]
keys({a = 1, b = 2})                       # ["a", "b"]
values({a = 1, b = 2})                     # [1, 2]
zipmap(["a", "b"], [1, 2])                # {a = 1, b = 2}
distinct(["a", "b", "a"])                  # ["a", "b"]
concat(["a"], ["b"], ["c"])               # ["a", "b", "c"]
```

### Encoding Functions

```hcl
jsonencode({name = "test", port = 80})    # JSON string
jsondecode(file("config.json"))           # HCL object
base64encode("hello")                      # "aGVsbG8="
yamlencode({key = "value"})               # YAML string
```

### Filesystem Functions

```hcl
file("scripts/init.sh")                    # Read file contents
fileexists("config.json")                  # true/false
templatefile("template.tpl", { port = 80 }) # Render template
```

### Network Functions

```hcl
cidrsubnet("10.0.0.0/16", 8, 1)   # "10.0.1.0/24"
cidrhost("10.0.1.0/24", 5)         # "10.0.1.5"
cidrnetmask("10.0.0.0/16")         # "255.255.0.0"
```

---

## Comments

```hcl
# Single-line comment

// Also single-line (less common)

/*
  Multi-line
  comment
*/
```

---

## Key Takeaways

1. **Blocks are the structure** — everything is `type "labels" { body }`
2. **Types are strict** — Terraform validates types at plan time, catching errors early
3. **Expressions are powerful** — conditionals, for loops, and splats handle most logic needs
4. **Functions are built-in only** — no custom functions, but the standard library is comprehensive
5. **Dynamic blocks for variable repetition** — use when nested block count varies; prefer explicit blocks otherwise
6. **References create dependencies** — Terraform builds a dependency graph from your references automatically
