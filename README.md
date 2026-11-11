# Terragrunt: Recommended Usage Guide

## 1. Do you even need Terragrunt?

Your understanding is mostly right. Terragrunt is a thin wrapper around Terraform/OpenTofu. It does not replace it.

| Situation | Recommendation |
|---|---|
| One root module + `dev/qa/prod.tfvars`, state managed by **Terraform Cloud** (state + locking built in) | **Plain Terraform is enough.** No Terragrunt. |
| One root module + tfvars, state in **S3** | Plain Terraform works, but you must pass a different backend key per env (`-backend-config`) or use workspaces. This is where Terragrunt starts to help. |
| **Many modules** (vpc, eks, rds, iam ...) x **many envs** (dev, qa, prod) | **Use Terragrunt.** It removes duplicated backend/provider blocks and wires module dependencies. |

What Terragrunt actually gives you:

- **DRY backend config**: one `remote_state` block generates the S3 backend for every module and env, with an automatic unique state key per folder.
- **DRY provider config**: one `generate "provider"` block instead of a `provider.tf` copied everywhere.
- **Auto-creates** the S3 bucket (and lock table, if used) on first run.
- **Dependencies** between modules (`dependency` blocks, e.g. EKS reads VPC outputs) and `run --all` to apply them in order.
- **Small state files**: one state per module per env, so blast radius is small.

> Note: Terraform 1.10+ supports **native S3 locking** (`use_lockfile = true`), so DynamoDB is no longer required for new setups. Use DynamoDB only if you are on an older Terraform version.

---

## 2. Your current structure (works, but state is the weak point)

```
project/
├── main.tf
├── provider.tf
├── variables.tf
├── outputs.tf
├── locals.tf
├── data.tf
└── Env/
    ├── dev.tfvars
    ├── qa.tfvars
    └── prod.tfvars
```

Plain Terraform usage:

```bash
terraform init -backend-config="key=dev/terraform.tfstate"
terraform plan  -var-file=Env/dev.tfvars
terraform apply -var-file=Env/dev.tfvars
```

Risk: if you forget to change the backend key (or run `init -reconfigure`), you can plan/apply dev values against prod state. Terragrunt removes that risk because the state key comes from the folder path.

---

## 3. Recommended Terragrunt structure

Separate **what to deploy** (modules) from **where/how to deploy it** (live config).

```
infra/
├── modules/                       # Your existing Terraform code, unchanged
│   └── app/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── locals.tf
│       └── data.tf                # (no provider.tf / backend here; Terragrunt generates it)
│
└── live/
    ├── root.hcl                   # Single reference file: backend + provider + common inputs
    ├── dev/
    │   ├── env.hcl                # env-level values (name, region, account)
    │   └── app/
    │       └── terragrunt.hcl     # points to modules/app + dev inputs
    ├── qa/
    │   ├── env.hcl
    │   └── app/
    │       └── terragrunt.hcl
    └── prod/
        ├── env.hcl
        └── app/
            └── terragrunt.hcl
```

When you add more modules (vpc, eks, rds), add a sibling folder under each env:

```
live/dev/
├── env.hcl
├── vpc/terragrunt.hcl
├── eks/terragrunt.hcl
└── rds/terragrunt.hcl
```

Key idea: **every env folder calls the same `root.hcl`**, so backend and provider code is written once.

---

## 4. Files

### 4.1 `live/root.hcl` (the single reference file)

```hcl
locals {
  # Read env.hcl from the closest parent folder (dev/qa/prod)
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))

  env        = local.env_vars.locals.env
  aws_region = local.env_vars.locals.aws_region
  account_id = local.env_vars.locals.account_id
}

# Remote state in S3. The key is built from the folder path, so it is unique per module per env.
remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket       = "my-company-tfstate-${local.account_id}-${local.aws_region}"
    key          = "${path_relative_to_include()}/terraform.tfstate"
    region       = local.aws_region
    encrypt      = true
    use_lockfile = true            # native S3 locking (Terraform >= 1.10)

    # If you are on Terraform < 1.10, use DynamoDB locking instead:
    # dynamodb_table = "my-company-tf-locks"
  }
}

# Generate provider.tf for every module, so you never copy it again.
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"

  default_tags {
    tags = {
      Environment = "${local.env}"
      ManagedBy   = "terragrunt"
    }
  }
}
EOF
}

# Inputs available to every module in every env
inputs = {
  environment = local.env
  region      = local.aws_region
}
```

### 4.2 `live/dev/env.hcl`

```hcl
locals {
  env        = "dev"
  aws_region = "ap-south-1"
  account_id = "111111111111"
}
```

Repeat for `qa` and `prod` with their own values.

### 4.3 `live/dev/app/terragrunt.hcl`

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "../../../modules/app"
}

# Values that replace your old dev.tfvars
inputs = {
  instance_type = "t3.small"
  min_size      = 1
  max_size      = 2
}
```

`qa/app/terragrunt.hcl` and `prod/app/terragrunt.hcl` are identical except for `inputs` (and prod can pin a module version, see below).

### 4.4 Module `variables.tf` must declare the inputs

```hcl
variable "environment"   { type = string }
variable "region"        { type = string }
variable "instance_type" { type = string }
variable "min_size"      { type = number }
variable "max_size"      { type = number }
```

Remove `provider.tf` and any `terraform { backend ... }` block from the module, because Terragrunt generates them.

---

## 5. Migrating from your current structure

1. Create `infra/modules/app/` and move `main.tf`, `variables.tf`, `outputs.tf`, `locals.tf`, `data.tf` into it.
2. Delete `provider.tf` (now generated by `root.hcl`).
3. Convert each `Env/*.tfvars` into the `inputs = { ... }` block of the matching `live/<env>/app/terragrunt.hcl`.
4. Create `root.hcl` and `env.hcl` files as shown above.
5. **If you already have state**, migrate it. Do not just run apply, or Terraform will try to recreate everything:
   ```bash
   cd live/dev/app
   terragrunt init -migrate-state
   terragrunt plan        # should show NO changes
   ```
6. Repeat for qa and prod. Always confirm that `plan` is clean before moving on.

---

## 6. Daily commands

Run from inside the module folder:

```bash
cd live/dev/app

terragrunt init
terragrunt plan
terragrunt apply
terragrunt destroy
```

Run for a whole environment (all modules, dependency order respected):

```bash
cd live/dev

terragrunt run --all plan        # Terragrunt >= 0.73
terragrunt run --all apply

# Older Terragrunt versions:
# terragrunt run-all plan
# terragrunt run-all apply
```

Useful extras:

```bash
terragrunt plan -out=tfplan                  # save a plan
terragrunt hcl format                        # format .hcl files (older: hclfmt)
terragrunt output                            # module outputs
terragrunt run --all plan --non-interactive  # CI usage
```

---

## 7. Dependencies between modules

Example: EKS needs VPC outputs.

```hcl
# live/dev/eks/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "../../../modules/eks"
}

dependency "vpc" {
  config_path = "../vpc"

  # Lets `plan` work before the VPC exists
  mock_outputs = {
    vpc_id          = "vpc-00000000"
    private_subnets = ["subnet-00000000"]
  }
  mock_outputs_allowed_terraform_commands = ["init", "validate", "plan"]
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnets
}
```

`terragrunt run --all apply` will apply `vpc` first, then `eks`.

---

## 8. Best practices checklist

- **Pin versions.** Add a `versions.tf` (or generate one) with `required_version` and provider versions. Commit `.terraform.lock.hcl`.
- **One state per module per env.** Never put everything in one state.
- **Use `root.hcl`, not a root `terragrunt.hcl`.** Newer Terragrunt versions recommend this to avoid confusion with child files.
- **Version your modules.** Keep modules in a separate repo and reference a tag, so prod only changes when you bump the tag:
  ```hcl
  terraform {
    source = "git::git@github.com:my-org/terraform-modules.git//app?ref=v1.4.0"
  }
  ```
  Promote the same tag dev -> qa -> prod.
- **Separate AWS accounts per env** (or at least separate state buckets). Set the account via `env.hcl`.
- **Encrypt and protect the state bucket**: versioning on, encryption on, public access blocked, restricted IAM.
- **Never commit secrets.** Use AWS Secrets Manager / SSM Parameter Store, or `sops`, not plain values in `inputs`.
- **Use `mock_outputs`** for dependencies so `plan` and `validate` work in CI.
- **Always `plan` before `apply`** in CI, with manual approval for prod.
- **Keep `.terragrunt-cache/` and `.terraform/` in `.gitignore`.**

`.gitignore`:

```
.terragrunt-cache/
.terraform/
*.tfstate
*.tfstate.backup
tfplan
```

---

## 9. Terragrunt vs Terraform Cloud

| | Terragrunt (+ S3) | Terraform Cloud |
|---|---|---|
| State storage and locking | S3 (+ native lock or DynamoDB), you manage it | Built in |
| Per-env config | Folder + `inputs` | Workspaces + variables / tfvars |
| DRY backend/provider | Yes, via `root.hcl` | Not needed (managed) |
| Module dependency ordering | `dependency` blocks, `run --all` | Run triggers between workspaces |
| Cost | Free (open source) | Free tier, then paid |

Pick **one** model for state and locking. Do not combine Terraform Cloud's backend with Terragrunt's `remote_state` for the same module.

---

## 10. Quick decision summary

- Single root module + tfvars + Terraform Cloud -> **plain Terraform**, no Terragrunt.
- Multiple modules and multiple envs with S3 state -> **Terragrunt**, with one `root.hcl`, one `env.hcl` per env, and a thin `terragrunt.hcl` per module per env.
