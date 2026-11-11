# Terragrunt - Terraform Wrapper for DRY, Maintainable Infrastructure

> **Version**: Terragrunt >= 1.0 (CLI syntax in this guide is the stable 1.x CLI) with Terraform or OpenTofu >= 1.0  
> Native S3 state locking (`use_lockfile`) needs Terraform / OpenTofu >= 1.10  
> **Last Updated**: 2026-11-11

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

**Terragrunt** is a thin wrapper around [Terraform](https://www.terraform.io/) / [OpenTofu](https://opentofu.org/) that provides:

| Feature | Description |
|---------|-------------|
| **DRY Configuration** | Define common settings once, inherit everywhere |
| **Remote State Configuration** | Generates the S3/GCS/Azure backend block for every module, with a unique state key per folder |
| **Backend Bootstrap** | `terragrunt backend bootstrap` creates the state bucket (and lock table, if configured) |
| **Dependency Management** | Track dependencies between Terraform modules and run them in order |
| **Multiple Environments** | Same code, different configs (dev/staging/prod) |
| **Hooks & Error Handling** | Pre/post execution hooks |
| **Generate Files** | Auto-generate provider configs, backend configs |

> **Note on locking:** Terragrunt does not implement locking itself. Locking is done by the Terraform backend (S3 native lock file, or DynamoDB on older setups). Terragrunt only generates the backend config and can create the bucket/table for you.

---

## Why Terragrunt?

### Problem Without Terragrunt

```
❌ Duplicated backend config in every module
❌ Copy-pasted provider settings
❌ Backend key passed by hand per environment (easy to point dev values at prod state)
❌ No easy way to share variables across environments
```

### Solution With Terragrunt

```
✅ Single source of truth for backend/providers
✅ Hierarchical variable inheritance
✅ Automatic state isolation per module per environment
✅ Explicit dependency graph
```

### Do You Actually Need It?

| Situation | Recommendation |
|-----------|----------------|
| One root module + `dev/qa/prod.tfvars`, state in **Terraform Cloud** (state + locking built in) | **Plain Terraform is enough.** No Terragrunt. |
| One root module + tfvars, state in **S3** | Plain Terraform works (`-backend-config` per env), but it is easy to make mistakes. Terragrunt is a good fit. |
| **Many modules** (vpc, eks, rds, ...) x **many environments** | **Use Terragrunt.** |

Pick **one** model for state. Do not combine Terraform Cloud's backend with Terragrunt's `remote_state` for the same module.

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

### Pin the Version for Your Team (Recommended)

Use a version manager such as [mise](https://mise.jdx.dev/) or [tenv](https://github.com/tofuutils/tenv) so everyone (and CI) uses the same Terragrunt and Terraform versions.

### Verify Installation
```bash
terragrunt --version
# terragrunt version v1.x.x
```

### Old vs New CLI Syntax

Terragrunt 1.0 made the redesigned CLI stable. If you have older scripts, migrate them:

| Old (deprecated) | New |
|------------------|-----|
| `terragrunt run-all plan` | `terragrunt run --all plan` |
| `terragrunt hclfmt` | `terragrunt hcl fmt` |
| `terragrunt hclvalidate` | `terragrunt hcl validate` |
| `terragrunt validate-inputs` | `terragrunt hcl validate --inputs` |
| `terragrunt graph-dependencies` | `terragrunt dag graph` |
| `terragrunt render-json` | `terragrunt render --json` |
| `terragrunt-info` | `terragrunt info print` |
| `--terragrunt-non-interactive` | `--non-interactive` |
| `--terragrunt-working-dir` | `--working-dir` |
| `TERRAGRUNT_*` env vars | `TG_*` env vars |

---

## Project Structure Best Practices

### Recommended Directory Layout

```
infrastructure/
├── root.hcl                       # Root configuration (shared by every module)
├── common.hcl                     # Common variables
├── environments/
│   ├── dev/
│   │   ├── env.hcl               # Environment-specific settings
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl    # Module (unit) configuration
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

> **Why `root.hcl` and not a root `terragrunt.hcl`?**  
> A root file named `terragrunt.hcl` is no longer the recommended pattern. Naming it `root.hcl` avoids confusion with the per-module `terragrunt.hcl` files, and Terragrunt's strict mode can flag the old pattern.

### Small Project: One Module + tfvars per Environment

If your project is a single Terraform root module (`main.tf`, `variables.tf`, `outputs.tf`, ...) with `dev/qa/prod.tfvars`, use the same layout with a single module called `app`:

```
infrastructure/
├── root.hcl
├── common.hcl
├── modules/
│   └── app/                       # your existing main.tf, variables.tf, outputs.tf, locals.tf, data.tf
└── environments/
    ├── dev/
    │   ├── env.hcl
    │   └── app/
    │       ├── terragrunt.hcl
    │       └── app.tfvars         # replaces Env/dev.tfvars
    ├── qa/
    │   └── ...
    └── prod/
        └── ...
```

Remove `provider.tf` and any `backend` block from the module. Terragrunt generates them.

> A single `terragrunt.hcl` next to a flat set of `dev/staging/prod.tfvars` files cannot give each environment its own state. Each environment needs its own folder.

---

## Configuration Files Explained

| File | Purpose | Scope |
|------|---------|-------|
| `root.hcl` | Shared backend, provider, common inputs | Root level |
| `terragrunt.hcl` | Module (unit) config: source + inputs | Per-module |
| `terragrunt.hcl.json` | JSON alternative | Per-module |
| `common.hcl` | Shared variables | Root level |
| `env.hcl` | Environment variables (name, region, account) | Per-environment |
| `*.tfvars` | Terraform variables | Passed to Terraform via `-var-file` |
| `*.tfvars.json` | JSON format variables | Passed to Terraform via `-var-file` |

> `find_in_parent_folders("env.hcl")` searches the **parent** folders of the current module. So `env.hcl` must live above the module folder (for example `environments/dev/env.hcl` for `environments/dev/vpc/terragrunt.hcl`).

---

## Handling tfvars Files

### The Problem: Multiple Variable Sources

When you have:
- shared values (project name, owner, tags)
- environment values (dev vs prod)
- sizing values per module (`*.tfvars`)
- secrets

**How do they reach Terraform?**

### Method 1: Using `inputs` (Recommended)

`inputs` are passed to Terraform as `TF_VAR_*` environment variables. No tfvars files are needed, and values can be computed with HCL.

```hcl
# environments/dev/vpc/terragrunt.hcl
locals {
  common = read_terragrunt_config(find_in_parent_folders("common.hcl"))
  env    = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

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
    { Environment = local.env.locals.environment }
  )
}
```

Inputs defined in the root (`root.hcl`) are merged with the inputs of each module. The module's values win on conflict.

### Method 2: Keep Your tfvars Files with `extra_arguments`

If you already have `*.tfvars` files, keep them and let Terragrunt pass them with `-var-file`:

```hcl
# environments/dev/vpc/terragrunt.hcl
terraform {
  extra_arguments "env_vars" {
    commands = get_terraform_commands_that_need_vars()

    # Missing file = error
    required_var_files = [
      "${get_terragrunt_dir()}/vpc.tfvars"
    ]

    # Missing file = ignored (good for local-only secrets)
    optional_var_files = [
      "${get_terragrunt_dir()}/secrets.tfvars"
    ]
  }
}
```

Use `required_var_files` / `optional_var_files` instead of building `-var-file=...` strings by hand, so you do not need `fileexists()` tricks.

### Variable Precedence (Important)

| Order (low to high) | Source |
|---------------------|--------|
| 1 | `default` in `variables.tf` |
| 2 | `inputs` (passed as `TF_VAR_*`) |
| 3 | `-var-file` files, in the order given (later wins) |

So a value in a tfvars file **overrides** the same key in `inputs`. Do not define the same variable in both places. Pick one source per variable.

### Method 3: Generate a tfvars File

Rarely needed, but useful when a tool expects a file on disk:

```hcl
# terragrunt.hcl
generate "tfvars" {
  path      = "terragrunt.auto.tfvars"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
environment = "${local.env.locals.environment}"
region      = "${local.env.locals.region}"
project     = "${local.common.locals.project_name}"
EOF
}
```

---

## Complete Working Example

### File Structure

```
infrastructure/
├── root.hcl                    # Root config (backend, provider, shared inputs)
├── common.hcl                  # Shared variables
├── modules/
│   └── vpc/                    # main.tf, variables.tf, outputs.tf
└── environments/
    ├── dev/
    │   ├── env.hcl
    │   └── vpc/
    │       ├── terragrunt.hcl
    │       └── vpc.tfvars
    └── prod/
        ├── env.hcl
        └── vpc/
            ├── terragrunt.hcl
            └── vpc.tfvars
```

### 1. Root `root.hcl`

```hcl
# infrastructure/root.hcl

locals {
  common = read_terragrunt_config(find_in_parent_folders("common.hcl"))
  env    = read_terragrunt_config(find_in_parent_folders("env.hcl"))

  environment = local.env.locals.environment
  region      = local.env.locals.region
}

# Generate the backend block. State key = folder path, so it is unique per module per env.
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket       = "my-terraform-state-${get_aws_account_id()}"
    key          = "${path_relative_to_include()}/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true   # native S3 locking (Terraform / OpenTofu >= 1.10)

    # On older Terraform versions use DynamoDB locking instead:
    # dynamodb_table = "terraform-locks"
  }
}

# Generate provider configuration
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.region}"

  default_tags {
    tags = {
      ManagedBy   = "terragrunt"
      Project     = "${local.common.locals.project_name}"
      Environment = "${local.environment}"
    }
  }
}
EOF
}

# Inputs that every module in every environment inherits
inputs = {
  project_name = local.common.locals.project_name
  owner        = local.common.locals.owner
  environment  = local.environment
  region       = local.region
}
```

> The generated `provider.tf` takes its values from Terragrunt locals, so your modules do not need `region`, `project_name` or `environment` variables just for the provider. Declare only the variables the module itself uses, otherwise Terraform prints "undeclared variable" warnings.

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

### 3. `env.hcl` (Per Environment)

```hcl
# infrastructure/environments/dev/env.hcl
locals {
  environment = "dev"
  region      = "us-east-1"
}
```

```hcl
# infrastructure/environments/prod/env.hcl
locals {
  environment = "prod"
  region      = "us-east-1"
}
```

### 4. Module tfvars Files

```hcl
# infrastructure/environments/dev/vpc/vpc.tfvars
vpc_cidr           = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]
```

```hcl
# infrastructure/environments/prod/vpc/vpc.tfvars
vpc_cidr           = "10.1.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
```

Keep one tfvars file per module, containing only the variables that module declares. `environment`, `region`, `project_name` and `owner` already arrive through the root `inputs`, so do not repeat them here.

### 5. Module-Level `terragrunt.hcl`

```hcl
# infrastructure/environments/dev/vpc/terragrunt.hcl

# Include root configuration
include "root" {
  path   = find_in_parent_folders("root.hcl")
  expose = true  # Make root locals available as include.root.locals
}

# Terraform source
terraform {
  source = "../../../modules//vpc"

  # Module-specific tfvars
  extra_arguments "env_vars" {
    commands           = get_terraform_commands_that_need_vars()
    required_var_files = ["${get_terragrunt_dir()}/vpc.tfvars"]
  }
}

# Extra inputs for this module (merged with the root inputs)
inputs = {
  tags = merge(
    include.root.locals.common.locals.common_tags,
    {
      Environment = include.root.locals.environment
      Module      = "vpc"
    }
  )
}
```

The prod file is identical, except it lives in `environments/prod/vpc/` and uses its own `vpc.tfvars`.

### 6. Running Commands

```bash
# First time only: create the S3 state bucket (and lock table, if configured)
cd infrastructure/environments/dev/vpc
terragrunt backend bootstrap

# Plan / apply one module
terragrunt plan
terragrunt apply

# Plan every module in dev (dependency order)
cd ../
terragrunt run --all plan

# Apply every module in dev
terragrunt run --all apply
```

> Do not pass relative `-var-file=../../x.tfvars` paths on the command line. Terragrunt runs Terraform inside a cache folder, so relative paths break. Define var files in `extra_arguments` using `get_terragrunt_dir()` instead.

> **Backend bootstrap:** Terragrunt no longer creates the state bucket automatically on every run. Run `terragrunt backend bootstrap` once, or opt in to the old behavior with `--backend-bootstrap` (or `TG_BACKEND_BOOTSTRAP=true`) in CI.

---

## Advanced Patterns

### Pattern 1: Multiple tfvars with Precedence

```hcl
# terragrunt.hcl
terraform {
  extra_arguments "vars" {
    commands = get_terraform_commands_that_need_vars()

    # Order matters: later files override earlier ones
    required_var_files = [
      "${get_parent_terragrunt_dir("root")}/common.tfvars",
      "${get_parent_terragrunt_dir("root")}/region.tfvars",
      "${get_terragrunt_dir()}/override.tfvars"   # Module-specific
    ]
  }
}
```

### Pattern 2: Optional tfvars (e.g. local secrets)

```hcl
terraform {
  extra_arguments "vars" {
    commands = get_terraform_commands_that_need_vars()

    required_var_files = [
      "${get_terragrunt_dir()}/vpc.tfvars"
    ]

    # Used only if the file exists. Never commit this file.
    optional_var_files = [
      "${get_terragrunt_dir()}/secrets.tfvars"
    ]
  }
}
```

### Pattern 3: Dependency Management

```hcl
# infrastructure/environments/dev/eks/terragrunt.hcl

dependency "vpc" {
  config_path = "../vpc"

  # Mock outputs for plan/validate before the VPC exists
  mock_outputs = {
    vpc_id          = "vpc-12345678"
    private_subnets = ["subnet-123", "subnet-456"]
  }
  mock_outputs_allowed_terraform_commands = ["init", "validate", "plan"]
}

inputs = {
  vpc_id          = dependency.vpc.outputs.vpc_id
  private_subnets = dependency.vpc.outputs.private_subnets
}
```

### Pattern 4: Secrets and Hooks

Prefer reading secrets at runtime (AWS Secrets Manager / SSM Parameter Store) inside Terraform. If you use SOPS, decrypt with the built-in function instead of decrypting and re-encrypting files in hooks:

```hcl
locals {
  secrets = yamldecode(sops_decrypt_file("${get_terragrunt_dir()}/secrets.enc.yaml"))
}

inputs = {
  db_password = local.secrets.db_password
}
```

> Values passed this way can still end up in the Terraform state. Mark the variable `sensitive = true` and protect the state bucket.

Hooks are good for checks and housekeeping:

```hcl
terraform {
  before_hook "tflint" {
    commands = ["plan"]
    execute  = ["tflint"]
  }

  after_hook "notify" {
    commands     = ["apply"]
    execute      = ["echo", "apply finished"]
    run_on_error = true
  }
}
```

### Pattern 5: Promote Module Versions per Environment

Keep modules in a separate Git repo and pin a tag per environment. Dev can run a newer tag than prod.

```hcl
terraform {
  source = "git::git@github.com:my-org/terraform-modules.git//vpc?ref=v1.4.0"
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
| `terragrunt backend bootstrap` | Create the state bucket / lock table |
| `terragrunt run --all plan` | Plan all modules in tree |
| `terragrunt run --all apply` | Apply all modules |
| `terragrunt run -- <cmd>` | Run any other Terraform command (for example `workspace ls`) |
| `terragrunt hcl fmt` | Format all .hcl files |
| `terragrunt hcl validate --inputs` | Check inputs match variables |
| `terragrunt dag graph` | Show dependency graph |
| `terragrunt list` / `terragrunt find` | List the units in the repo (human / script friendly) |
| `terragrunt render --json` | Output JSON representation of the config |

### Useful Flags

```bash
# Non-interactive mode (CI)
terragrunt run --all apply --non-interactive

# Working directory
terragrunt apply --working-dir=environments/dev/vpc

# Config file
terragrunt apply --config=custom.hcl

# Debug logging
terragrunt apply --log-level debug

# Pass flags through to Terraform (after --)
terragrunt run -- apply -auto-approve
```

---

## Best Practices Checklist

### ✅ Configuration
- [ ] Name the shared file `root.hcl` and use `include "root" { path = find_in_parent_folders("root.hcl") }`
- [ ] Always give `include` blocks a name
- [ ] Keep environment-specific values in `env.hcl`
- [ ] Prefer `inputs` over tfvars/`extra_arguments`; if you use both, never define the same variable twice
- [ ] Use `locals` for computed values
- [ ] Use `generate` blocks for provider/backend configs

### ✅ State Management
- [ ] Always use remote state (S3/GCS/Azure)
- [ ] Enable state locking (`use_lockfile` on S3, or DynamoDB on older Terraform)
- [ ] Use unique state keys per module: `${path_relative_to_include()}`
- [ ] Enable versioning on state buckets
- [ ] Run `terragrunt backend bootstrap` explicitly instead of relying on auto-creation

### ✅ Security
- [ ] Never commit secrets to version control (add `secrets.tfvars` to `.gitignore`)
- [ ] Use SOPS, Vault, or AWS Secrets Manager
- [ ] Encrypt state files at rest and block public access on the bucket
- [ ] Use IAM roles with least privilege

### ✅ Code Quality
- [ ] Run `terragrunt hcl fmt` before committing
- [ ] Use `terragrunt hcl validate --inputs` to catch typos
- [ ] Document module inputs/outputs
- [ ] Use consistent naming conventions

### ✅ Team Workflow
- [ ] Use `dependency` blocks with `mock_outputs`
- [ ] Run `terragrunt run --all validate` in CI
- [ ] Use `terragrunt dag graph` to visualize
- [ ] Pin Terragrunt, Terraform and provider versions (and commit `.terraform.lock.hcl`)
- [ ] Always plan before apply; require approval for prod

Suggested `.gitignore`:

```
.terragrunt-cache/
.terraform/
*.tfstate
*.tfstate.backup
secrets.tfvars
```

---

## Troubleshooting

### Issue: "Variables not found" / "undeclared variable" warnings

**Solution**: Check var file paths, precedence, and that the module declares every variable you pass
```bash
# Debug variable loading
terragrunt plan --log-level debug 2>&1 | grep -i var-file

# Check inputs against the module's variables
terragrunt hcl validate --inputs
```

### Issue: Wrong value used (tfvars ignored `inputs`)

**Solution**: `-var-file` values override `inputs`. Remove the duplicate definition from one of them.

### Issue: "State file locked"

**Solution**: Force unlock (use carefully, only if you are sure no run is active)
```bash
terragrunt run -- force-unlock <LOCK_ID>
```

### Issue: "Cycle in dependency graph"

**Solution**: Review `dependency` blocks for circular references
```bash
terragrunt dag graph | dot -Tsvg > graph.svg
```

### Issue: "Backend configuration changed"

**Solution**: Reinitialize
```bash
terragrunt run -- init -reconfigure
```

### Issue: "S3 bucket does not exist" on first run

**Solution**: Create the backend resources
```bash
terragrunt backend bootstrap
```

### Issue: Moving existing state into Terragrunt

**Solution**: Migrate state, then confirm the plan is clean before applying
```bash
cd environments/dev/app
terragrunt run -- init -migrate-state
terragrunt plan        # must show NO changes
```

---

## Resources

- [Official Documentation](https://docs.terragrunt.com/)
- [GitHub Repository](https://github.com/gruntwork-io/terragrunt)
- [CLI Redesign Migration Guide](https://docs.terragrunt.com/migrate/cli-redesign/)
- [Community Slack](https://gruntwork-community.slack.com/)

---

## License

This README is provided as a reference implementation. Adapt to your organization's needs.

---

**Happy Infrastructure Managing! 🚀**
