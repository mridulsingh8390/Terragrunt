# Terragrunt - Terraform Wrapper for DRY, Maintainable Infrastructure

> **Version**: Compatible with Terraform >= 1.0 and Terragrunt >= 0.50  
> **Last Updated**: 2026-11-09

---

## 📋 Table of Contents

1. [What is Terragrunt?](#what-is-terragrunt)
2. [Why Terragrunt?](#why-terragrunt)
3. [Installation](#installation)
4. [Project Structure Best Practices](#project-structure-best-practices)
5. [Configuration Files Explained](#configuration-files-explained)
6. [Handling tfvars Files](#handling-tfvars-files)
7. [Complete Working Example](#complete-working-example)
8. [Advanced Patterns](#advanced-patterns)
9. [Common Commands](#common-commands)
10. [Best Practices Checklist](#best-practices-checklist)
11. [Troubleshooting](#troubleshooting)

---

## What is Terragrunt?

**Terragrunt** is a thin wrapper around [Terraform](https://www.terraform.io/) that provides:

| Feature | Description |
|---------|-------------|
| **DRY Configuration** | Define common settings once, inherit everywhere |
| **Remote State Management** | Automatic S3/GCS/Azure backend configuration |
| **Locking** | Built-in DynamoDB/GCP locking for state files |
| **Dependency Management** | Track dependencies between Terraform modules |
| **Multiple Environments** | Same code, different configs (dev/staging/prod) |
| **Hooks & Error Handling** | Pre/post execution hooks |
| **Generate Files** | Auto-generate provider configs, backend configs |

---

## Why Terragrunt?

### Problem Without Terragrunt

```
❌ Duplicated backend config in every module
❌ Copy-pasted provider settings
❌ Manual state file management
❌ No easy way to share variables across environments
```

### Solution With Terragrunt

```
✅ Single source of truth for backend/providers
✅ Hierarchical variable inheritance
✅ Automatic state isolation per environment
✅ Explicit dependency graph
```

---

## Installation

### macOS
```bash
brew install terragrunt
```

### Linux
```bash
# Download latest release
wget https://github.com/gruntwork-io/terragrunt/releases/latest/download/terragrunt_linux_amd64
chmod +x terragrunt_linux_amd64
sudo mv terragrunt_linux_amd64 /usr/local/bin/terragrunt
```

### Windows
```powershell
# Using Chocolatey
choco install terragrunt

# Or download binary from GitHub releases
```

### Verify Installation
```bash
terragrunt --version
# terragrunt version v0.67.0
```

---

## Project Structure Best Practices

### Recommended Directory Layout

```
infrastructure/
├── terragrunt.hcl                 # Root configuration (shared)
├── common.hcl                     # Common variables
├── account.hcl                    # Account-level settings
├── environments/
│   ├── dev/
│   │   ├── env.hcl               # Environment-specific settings
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl    # Module configuration
│   │   ├── eks/
│   │   │   └── terragrunt.hcl
│   │   └── rds/
│   │       └── terragrunt.hcl
│   ├── staging/
│   │   ├── env.hcl
│   │   └── ...
│   └── prod/
│       ├── env.hcl
│       └── ...
└── modules/                       # Reusable Terraform modules
    ├── vpc/
    ├── eks/
    └── rds/
```

### Alternative: Flat Structure (Small Projects)

```
infrastructure/
├── terragrunt.hcl
├── dev.tfvars
├── staging.tfvars
├── prod.tfvars
└── modules/
```

---

## Configuration Files Explained

| File | Purpose | Scope |
|------|---------|-------|
| `terragrunt.hcl` | Main Terragrunt config | Per-module or root |
| `terragrunt.hcl.json` | JSON alternative | Per-module |
| `common.hcl` | Shared variables | Root level |
| `env.hcl` | Environment variables | Per-environment |
| `account.hcl` | Account/region settings | Root level |
| `*.tfvars` | Terraform variables | Passed to Terraform |
| `*.tfvars.json` | JSON format variables | Passed to Terraform |

---

## Handling tfvars Files

### The Problem: Multiple Variable Sources

When you have:
- `common.tfvars` (shared variables)
- `dev.tfvars` (environment-specific)
- `secrets.tfvars` (sensitive data)
- `terragrunt.hcl` (Terragrunt config)

**How does Terragrunt merge them?**

### Solution: Hierarchical Variable Loading

Terragrunt uses **layered configuration** with `read_terragrunt_config()`:

```hcl
# terragrunt.hcl (root)
locals {
  # Load common variables
  common = read_terragrunt_config("${get_parent_terragrunt_dir()}/common.hcl")

  # Load environment variables
  env = read_terragrunt_config("${get_parent_terragrunt_dir()}/${path_relative_to_include()}/env.hcl")

  # Merge all variables (later overrides earlier)
  merged_vars = merge(
    local.common.locals,
    local.env.locals
  )
}

# Pass to Terraform as tfvars
terraform {
  extra_arguments "common_vars" {
    commands = get_terraform_commands_that_need_vars()

    arguments = [
      "-var-file=${get_parent_terragrunt_dir()}/common.tfvars",
      "-var-file=${get_terragrunt_dir()}/${local.env.locals.environment}.tfvars"
    ]
  }
}
```

### Method 1: Using `extra_arguments` (Recommended)

```hcl
# terragrunt.hcl
terraform {
  # ... other config ...

  extra_arguments "variables" {
    commands = [
      "apply",
      "plan",
      "import",
      "push",
      "refresh",
      "destroy"
    ]

    arguments = [
      "-var-file=${get_parent_terragrunt_dir()}/common.tfvars",
      "-var-file=${get_terragrunt_dir()}/environment.tfvars",
      "-var-file=${get_terragrunt_dir()}/secrets.tfvars"  # Optional
    ]
  }
}
```

### Method 2: Using `tfvars` Files with `include`

```hcl
# Root terragrunt.hcl
locals {
  environment = "dev"
}

terraform {
  extra_arguments "vars" {
    commands = get_terraform_commands_that_need_vars()
    arguments = [
      "-var-file=${get_parent_terragrunt_dir()}/${local.environment}.tfvars"
    ]
  }
}
```

### Method 3: Programmatic Variable Generation

```hcl
# terragrunt.hcl
generate "tfvars" {
  path      = "terragrunt.auto.tfvars"
  if_exists = "overwrite"
  contents  = <<EOF
environment = "${local.env.locals.environment}"
region      = "${local.env.locals.region}"
project     = "${local.common.locals.project_name}"

# Computed values
tags = {
  Environment = "${local.env.locals.environment}"
  Project     = "${local.common.locals.project_name}"
  ManagedBy   = "terragrunt"
}
EOF
}
```

---

## Complete Working Example

### File Structure

```
infrastructure/
├── terragrunt.hcl              # Root config
├── common.hcl                  # Shared variables
├── common.tfvars               # Shared tfvars
├── dev.tfvars                  # Dev environment
├── prod.tfvars                 # Prod environment
└── environments/
    ├── dev/
    │   └── vpc/
    │       └── terragrunt.hcl
    └── prod/
        └── vpc/
            └── terragrunt.hcl
```

### 1. Root `terragrunt.hcl`

```hcl
# infrastructure/terragrunt.hcl

# Configure Terragrunt to automatically store tfstate files in S3
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite"
  }
  config = {
    bucket         = "my-terraform-state-${get_aws_account_id()}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Generate provider configuration
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      ManagedBy   = "terragrunt"
      Project     = var.project_name
      Environment = var.environment
    }
  }
}
EOF
}

# Load common configuration
locals {
  common = read_terragrunt_config("${get_parent_terragrunt_dir()}/common.hcl")
}

# Pass variables to Terraform
terraform {
  # Include common tfvars
  extra_arguments "common_vars" {
    commands = get_terraform_commands_that_need_vars()
    arguments = [
      "-var-file=${get_parent_terragrunt_dir()}/common.tfvars"
    ]
  }
}

# Inputs that all environments inherit
inputs = {
  project_name = local.common.locals.project_name
  owner        = local.common.locals.owner
}
```

### 2. `common.hcl` (Shared Variables)

```hcl
# infrastructure/common.hcl
locals {
  project_name = "my-awesome-project"
  owner        = "platform-team"

  common_tags = {
    Project   = "my-awesome-project"
    Owner     = "platform-team"
    ManagedBy = "terragrunt"
  }
}
```

### 3. `common.tfvars` (Shared tfvars)

```hcl
# infrastructure/common.tfvars
project_name = "my-awesome-project"
owner        = "platform-team"

common_tags = {
  Project   = "my-awesome-project"
  Owner     = "platform-team"
  ManagedBy = "terragrunt"
}
```

### 4. Environment tfvars Files

```hcl
# infrastructure/dev.tfvars
environment = "dev"
region      = "us-east-1"

vpc_cidr = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]

instance_type = "t3.micro"
min_size      = 1
max_size      = 3

db_instance_class = "db.t3.micro"
db_allocated_storage = 20
```

```hcl
# infrastructure/prod.tfvars
environment = "prod"
region      = "us-east-1"

vpc_cidr = "10.1.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

instance_type = "t3.large"
min_size      = 3
max_size      = 10

db_instance_class = "db.r5.large"
db_allocated_storage = 100

enable_deletion_protection = true
enable_multi_az = true
```

### 5. Module-Level `terragrunt.hcl`

```hcl
# infrastructure/environments/dev/vpc/terragrunt.hcl

# Include root configuration
include "root" {
  path   = find_in_parent_folders()
  expose = true  # Make root inputs available
}

# Load environment-specific config
locals {
  env = read_terragrunt_config("${get_parent_terragrunt_dir()}/../../dev.tfvars")
}

# Terraform source
terraform {
  source = "../../../modules//vpc"

  # Add environment-specific tfvars
  extra_arguments "env_vars" {
    commands = get_terraform_commands_that_need_vars()
    arguments = [
      "-var-file=${get_parent_terragrunt_dir()}/../../dev.tfvars"
    ]
  }
}

# Environment-specific inputs
inputs = {
  environment        = "dev"
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b"]

  tags = merge(
    include.root.inputs.common_tags,
    {
      Environment = "dev"
      Module      = "vpc"
    }
  )
}
```

### 6. Running Commands

```bash
# Plan dev environment
cd infrastructure/environments/dev/vpc
terragrunt plan

# Apply with explicit var file
terragrunt apply -var-file=../../../dev.tfvars

# Plan all environments
terragrunt run-all plan

# Apply with confirmation
terragrunt run-all apply
```

---

## Advanced Patterns

### Pattern 1: Multiple tfvars with Precedence

```hcl
# terragrunt.hcl
terraform {
  extra_arguments "vars" {
    commands = get_terraform_commands_that_need_vars()
    arguments = [
      # Order matters: later files override earlier ones
      "-var-file=${get_parent_terragrunt_dir()}/common.tfvars",
      "-var-file=${get_parent_terragrunt_dir()}/region.tfvars",
      "-var-file=${get_parent_terragrunt_dir()}/env.tfvars",
      "-var-file=${get_terragrunt_dir()}/override.tfvars"  # Module-specific
    ]
  }
}
```

### Pattern 2: Conditional tfvars Loading

```hcl
locals {
  env = get_env("ENV", "dev")

  # Only include secrets file if it exists
  secrets_file = fileexists("${get_terragrunt_dir()}/secrets.tfvars") ?     "-var-file=${get_terragrunt_dir()}/secrets.tfvars" : ""
}

terraform {
  extra_arguments "vars" {
    commands = get_terraform_commands_that_need_vars()
    arguments = compact([
      "-var-file=${get_parent_terragrunt_dir()}/common.tfvars",
      "-var-file=${get_parent_terragrunt_dir()}/${local.env}.tfvars",
      local.secrets_file
    ])
  }
}
```

### Pattern 3: Using `inputs` Instead of tfvars

```hcl
# terragrunt.hcl
locals {
  common = read_terragrunt_config("${get_parent_terragrunt_dir()}/common.hcl")
  env      = read_terragrunt_config("${get_parent_terragrunt_dir()}/env.hcl")
}

# Direct variable assignment (no tfvars needed)
inputs = {
  # From common.hcl
  project_name = local.common.locals.project_name
  owner        = local.common.locals.owner

  # From env.hcl
  environment = local.env.locals.environment
  region      = local.env.locals.region

  # Computed values
  name_prefix = "${local.common.locals.project_name}-${local.env.locals.environment}"

  tags = merge(
    local.common.locals.common_tags,
    {
      Environment = local.env.locals.environment
    }
  )
}
```

### Pattern 4: Dependency Management

```hcl
# infrastructure/environments/dev/eks/terragrunt.hcl

dependency "vpc" {
  config_path = "../vpc"

  # Mock outputs for plan phase
  mock_outputs = {
    vpc_id          = "vpc-12345678"
    private_subnets = ["subnet-123", "subnet-456"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  vpc_id          = dependency.vpc.outputs.vpc_id
  private_subnets = dependency.vpc.outputs.private_subnets
}
```

### Pattern 5: Hooks for Secrets Management

```hcl
terraform {
  # Fetch secrets before running Terraform
  before_hook "decrypt_secrets" {
    commands = ["apply", "plan"]
    execute  = ["sops", "-d", "--in-place", "secrets.enc.yaml"]
  }

  # Clean up after
  after_hook "encrypt_secrets" {
    commands = ["apply", "plan"]
    execute  = ["sops", "-e", "--in-place", "secrets.enc.yaml"]
    run_on_error = true
  }
}
```

---

## Common Commands

| Command | Description |
|---------|-------------|
| `terragrunt plan` | Show execution plan |
| `terragrunt apply` | Apply changes |
| `terragrunt destroy` | Destroy infrastructure |
| `terragrunt output` | Show outputs |
| `terragrunt validate` | Validate configuration |
| `terragrunt run-all plan` | Plan all modules in tree |
| `terragrunt run-all apply` | Apply all modules |
| `terragrunt hclfmt` | Format all .hcl files |
| `terragrunt validate-inputs` | Check inputs match variables |
| `terragrunt graph-dependencies` | Show dependency graph |
| `terragrunt render-json` | Output JSON representation |

### Useful Flags

```bash
# Non-interactive mode
terragrunt apply --terragrunt-non-interactive

# Working directory
terragrunt apply --terragrunt-working-dir=environments/dev/vpc

# Config file
terragrunt apply --terragrunt-config=custom.hcl

# Debug logging
terragrunt apply --terragrunt-log-level debug
```

---

## Best Practices Checklist

### ✅ Configuration
- [ ] Use `include` to inherit from parent `terragrunt.hcl`
- [ ] Keep environment-specific values in `env.hcl` or `*.tfvars`
- [ ] Use `locals` for computed values
- [ ] Prefer `inputs` over `extra_arguments` when possible
- [ ] Use `generate` blocks for provider/backend configs

### ✅ State Management
- [ ] Always use remote state (S3/GCS/Azure)
- [ ] Enable state locking (DynamoDB/Cloud Storage)
- [ ] Use unique state keys per module: `${path_relative_to_include()}`
- [ ] Enable versioning on state buckets

### ✅ Security
- [ ] Never commit secrets to version control
- [ ] Use SOPS, Vault, or AWS Secrets Manager
- [ ] Encrypt state files at rest
- [ ] Use IAM roles with least privilege

### ✅ Code Quality
- [ ] Run `terragrunt hclfmt` before committing
- [ ] Use `terragrunt validate-inputs` to catch typos
- [ ] Document module inputs/outputs
- [ ] Use consistent naming conventions

### ✅ Team Workflow
- [ ] Use `dependency` blocks with `mock_outputs`
- [ ] Run `terragrunt run-all validate` in CI
- [ ] Use `terragrunt graph-dependencies` to visualize
- [ ] Pin Terraform and provider versions

---

## Troubleshooting

### Issue: "Variables not found"

**Solution**: Check variable precedence and file paths
```bash
# Debug variable loading
terragrunt plan --terragrunt-log-level debug 2>&1 | grep -i var-file
```

### Issue: "State file locked"

**Solution**: Force unlock (use carefully)
```bash
terragrunt force-unlock <LOCK_ID>
```

### Issue: "Cycle in dependency graph"

**Solution**: Review `dependency` blocks for circular references
```bash
terragrunt graph-dependencies | dot -Tsvg > graph.svg
```

### Issue: "Backend configuration changed"

**Solution**: Reinitialize
```bash
terragrunt init -reconfigure
```

---

## Resources

- [Official Documentation](https://terragrunt.gruntwork.io/)
- [GitHub Repository](https://github.com/gruntwork-io/terragrunt)
- [Best Practices Guide](https://terragrunt.gruntwork.io/docs/features/best-practices/)
- [Community Slack](https://gruntwork-community.slack.com/)

---

## License

This README is provided as a reference implementation. Adapt to your organization's needs.

---

**Happy Infrastructure Managing! 🚀**
