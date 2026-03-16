# OpenTofu Development Guide for Claude Code

This guide defines how to work with OpenTofu code, emphasising multi-file coordination,
linting with tflint, and security scanning with checkov.

> **Adapted from ArthurClune/claude-md-examples terraform-CLAUDE.md for OpenTofu.**
> Key differences: CLI is `tofu` (not `terraform`), registry is `registry.opentofu.org`,
> and OpenTofu-exclusive features (state encryption, ephemeral resources, `enabled`
> meta-argument) are preferred where applicable.

---

## General Principles

- Always scan the entire module directory before making changes
- Check for dependencies across files: variables, outputs, data sources, and resources
  may be referenced anywhere
- When modifying a resource, search for all references to it across `.tf` and `.tofu` files
- Consider module boundaries and inter-module dependencies

---

## CLI: Always Use `tofu`, Never `terraform`

All commands use the `tofu` binary. Never use `terraform` CLI in this project.

```bash
tofu init
tofu validate
tofu fmt -recursive
tofu plan -out=plan.out
tofu apply plan.out
tofu destroy
tofu state mv <source> <destination>
tofu state list
tofu output
tofu workspace list
tofu workspace select <name>
```

> **SAFETY RULE: Claude MUST NOT ever run `tofu apply` or `tofu destroy` — LINT and CHECK ONLY.**

---

## Required Version Block

Use the `terraform {}` block (OpenTofu keeps this name for HCL compatibility — a `tofu {}`
block does not yet exist as of v1.11).

```hcl
terraform {
  required_version = ">= 1.8.0" # OpenTofu version — current stable: 1.11

  required_providers {
    aws = {
      source  = "hashicorp/aws"   # resolves via registry.opentofu.org by default
      version = "~> 5.0"
    }
  }
}
```

> The default registry for OpenTofu is `registry.opentofu.org`, not `registry.terraform.io`.
> Short-form sources like `hashicorp/aws` resolve correctly without a full registry path.

---

## File Extensions

OpenTofu supports both `.tf` and `.tofu` extensions.

- Use `.tf` as the default for all files (cross-compatible, tooling support is universal)
- Use `.tofu` only for files that contain OpenTofu-exclusive features (state encryption,
  ephemeral resources, `enabled` meta-argument)
- Never define the same resource or variable in both a `.tf` and `.tofu` file

---

## tflint Workflow

### Check tflint is installed

```bash
which tflint || echo "tflint not installed"
ls -la .tflint.hcl
```

### Run tflint

```bash
# Initialise (first time or when plugins change)
tflint --init

# Run on current directory
tflint

# More detailed output
tflint --format compact

# Run recursively on all modules
tflint --recursive
```

---

## checkov Security Scanning

### Check checkov is installed

```bash
which checkov || echo "checkov not installed"
python3 --version  # needs 3.7+
```

### Run checkov

```bash
# Scan current directory
checkov -d .

# Scan with specific framework
checkov -d . --framework terraform

# Output results in JSON for parsing
checkov -d . -o json

# Skip specific checks
checkov -d . --skip-check CKV_AWS_20,CKV_AWS_23

# Run only specific checks
checkov -d . --check CKV_AWS_*
```

---

## Development Workflow

### Before making changes

```bash
# Check current state
tofu init
tofu validate
tflint
checkov -d .

# Review existing structure
find . -name "*.tf" -o -name "*.tofu" | sort
grep -r "resource\|module\|data" --include="*.tf" --include="*.tofu" | head -20
```

### When making changes

```bash
# Example: changing a security group — find all references first
grep -r "aws_security_group\.example" --include="*.tf" --include="*.tofu"
grep -r "security_group_id" --include="*.tf" --include="*.tofu"
```

- Update systematically: Start with variables, then resources, then outputs
- Maintain consistency: Keep naming conventions and patterns

### After making changes

```bash
# Validate syntax
tofu fmt -recursive
tofu validate

# Check for issues
tflint --recursive
checkov -d .

# Plan to verify (READ ONLY — do not apply)
tofu plan -out=plan.out
```

---

## CI/CD Pre-commit Script

```bash
#!/bin/bash
set -e
tofu fmt -check -recursive
tofu init -backend=false
tofu validate
tflint --recursive
checkov -d . --quiet --compact
```

---

## OpenTofu-Exclusive Features

Use these OpenTofu features where applicable — they are preferred over Terraform equivalents.

### State Encryption (OpenTofu 1.7+)

Native state encryption — always configure for production environments. Never store
unencrypted state in remote backends for sensitive workloads.

```hcl
# In a .tofu file (OpenTofu-only feature)
terraform {
  encryption {
    key_provider "pbkdf2" "main" {
      passphrase = var.state_passphrase
    }
    method "aes_gcm" "main" {
      keys = key_provider.pbkdf2.main
    }
    state {
      method = method.aes_gcm.main
    }
  }
}
```

### Ephemeral Resources (OpenTofu 1.10+)

Use for secrets and credentials that must never be persisted to state.

```hcl
ephemeral "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db.id
}
```

### `enabled` Meta-argument (OpenTofu 1.11+)

Cleaner alternative to `count = 0` / `count = 1` for conditional resources.

```hcl
resource "aws_instance" "bastion" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    enabled = var.create_bastion
  }
}
```

### Provider-defined Functions (OpenTofu 1.7+)

Providers can expose callable functions directly in HCL — use when available rather
than wrapping logic in `local` expressions.

---

## Naming Conventions

- Resources: `<provider>_<service>_<descriptor>` e.g. `aws_s3_bucket_logs`
- Variables: snake_case, descriptive e.g. `vpc_cidr_block`
- Locals: snake_case e.g. `common_tags`
- Modules: lowercase hyphenated directory names e.g. `modules/vpc-baseline`

---

## Variable Validation

Always add validation blocks for variables that have constrained values:

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

---

## Multi-file Change Checklist

When making changes that span multiple files:

- [ ] No hardcoded secrets or credentials
- [ ] No magic numbers — extract to `locals` or variables
- [ ] Proper types declared on all variables (bool not string for flags)
- [ ] Validation blocks on constrained variables
- [ ] `sensitive = true` on outputs containing secrets
- [ ] Security baseline passes checkov
- [ ] Naming conventions followed throughout
- [ ] DRY — use `for_each` not `count` for multiple similar resources
- [ ] Documentation: all variables and outputs have `description`
- [ ] `tofu fmt -recursive` passes with no changes
- [ ] `tofu validate` passes
- [ ] `tflint --recursive` passes
- [ ] `checkov -d .` passes (or suppressions are justified)

---

## Lock File

The dependency lock file is `.terraform.lock.hcl` — OpenTofu intentionally kept this
name for compatibility. Commit it to version control.

```bash
# Update lock file for all platforms (run after provider version changes)
tofu providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64
```

---

## Registry

- OpenTofu default registry: `registry.opentofu.org`
- Terraform registry (`registry.terraform.io`) is **not** used in this project
- To override the registry URL (e.g. for airgapped environments):
  ```bash
  export TFREGISTRY_BASE_URL=https://registry.opentofu.org/
  ```
