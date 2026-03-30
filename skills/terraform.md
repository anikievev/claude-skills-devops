---
name: terraform
description: Write well-structured, production-grade Terraform code following module conventions, remote state, security, and least-privilege IAM best practices
---

Analyze the current project context and write production-grade Terraform code. If infrastructure already exists, review and improve it.

## Requirements

**Inspect the project first:**
- Identify the target cloud provider (AWS, GCP, Azure)
- Understand what infrastructure is being provisioned
- Check for existing `.tf` files, `terraform.tfvars`, or `backend.tf`

**File structure — always use this layout:**

```
.
├── main.tf          # Resource definitions
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── versions.tf      # Required providers and Terraform version constraints
├── backend.tf       # Remote state backend configuration
└── terraform.tfvars.example  # Example variable values (never commit real values)
```

For reusable code, use modules under `modules/<name>/` with the same file layout.

For multi-environment projects, prefer **separate state files per environment** over workspaces:

```
environments/
├── dev/
│   ├── main.tf       # calls ../modules/app
│   ├── backend.tf    # separate state file for dev
│   └── terraform.tfvars
├── staging/
└── production/
modules/
└── app/              # shared module, no environment logic
```

**Mandatory rules:**

### `versions.tf`
- Always pin Terraform version with `required_version = ">= X.Y, < X+1.0"`
- Always pin provider versions with `~>` constraints (e.g. `~> 5.0`)
- Never omit the `required_providers` block

### `variables.tf`
- Every variable must have: `description`, `type`, and either `default` or be explicitly required
- Use `validation` blocks for any variable that has constraints (e.g. allowed values, regex patterns)
- Mark sensitive variables with `sensitive = true`
- Use specific types — never use `any`; prefer `object({})` and `list(string)` over loose types

### `main.tf`
- Use `locals {}` for computed values and repeated expressions — never duplicate expressions
- Add a `tags = merge(local.common_tags, var.additional_tags)` pattern for all taggable resources
- Use `lifecycle` blocks where appropriate: `prevent_destroy = true` for stateful resources, `ignore_changes` for externally managed attributes
- Prefer data sources over hardcoded IDs for existing resources

### `backend.tf`
- Always configure remote state (S3 + DynamoDB for AWS, GCS for GCP, Azure Blob for Azure)
- Enable state locking and encryption at rest
- **Use separate state files per environment** (separate directories), not Terraform workspaces — workspaces share backend config and make it easy to accidentally apply dev changes to production; separate directories make the blast radius explicit

### `outputs.tf`
- Export all values that consuming modules or CI/CD pipelines will need
- Mark sensitive outputs with `sensitive = true`
- Always include `description` on every output

### `for_each` vs `count`
- Use `for_each` when resources have meaningful, stable identifiers (e.g. a map of bucket names, a set of IAM users) — removing one element does not destroy and recreate others
- Use `count` only for simple boolean on/off toggles: `count = var.enable_monitoring ? 1 : 0`
- Never use `count` to iterate over a list of strings where order matters — adding/removing an element shifts indices and triggers unwanted replacements
- Prefer `for_each = toset(var.names)` over `count = length(var.names)`

### Multi-region / multi-account providers
When resources span regions or accounts, use provider aliases — never duplicate the whole module:

```hcl
# versions.tf
provider "aws" {
  region = var.primary_region
}

provider "aws" {
  alias  = "replica"
  region = var.replica_region
}
```

Pass aliases into modules via `providers` argument:
```hcl
module "replication" {
  source    = "./modules/replication"
  providers = {
    aws         = aws
    aws.replica = aws.replica
  }
}
```

### IAM & Security
- Write least-privilege IAM policies — no `*` actions unless absolutely required and justified with a comment
- Never hardcode credentials, account IDs, or ARNs — use data sources and variables
- Enable encryption at rest and in transit for all supported resources
- Use `random_id` or `random_pet` for unique name suffixes where needed

### Naming conventions
- Use kebab-case for resource names: `my-service-prod`
- Prefix all resources with `${var.project}-${var.environment}-`
- Be explicit — avoid abbreviations that reduce readability

### Anti-patterns — never do these
- **God module** — a single module that provisions VPC, compute, database, IAM, and DNS together; split by domain
- **Hardcoded environment logic inside modules** — modules must be environment-agnostic; environment differences belong in the calling layer (`environments/prod/`)
- **`terraform.workspace` for environment isolation** — workspaces are for testing short-lived variants of the same config, not for prod/staging/dev separation
- **`count` on resource maps** — causes index-shift destroys; use `for_each`
- **`terraform apply` without a saved plan** — always `terraform plan -out=tfplan` then `terraform apply tfplan` in CI; never apply interactively in production
- **Storing state locally** — local `terraform.tfstate` has no locking, no history, and gets lost; always use remote backend from day one
- **Circular module dependencies** — module A calling module B calling module A; extract the shared resource into a third module

### Importing existing infrastructure
When bringing unmanaged resources under Terraform management, never write the resource block and `apply` — it will try to create a duplicate. Use the import workflow:

```hcl
# 1. Add an import block (Terraform 1.5+) — generates the resource config automatically
import {
  to = aws_s3_bucket.main
  id = "my-existing-bucket-name"
}
```

Then run:
```bash
# Generate the resource config from the live resource
terraform plan -generate-config-out=generated.tf

# Review generated.tf, clean it up, move it into main.tf
# Then import state without creating a new resource
terraform apply
```

For older Terraform versions: `terraform import aws_s3_bucket.main my-existing-bucket-name`

### CI/CD integration
On pull request — plan only, never apply:
```bash
terraform init -backend-config=environments/prod/backend.hcl
terraform validate
terraform plan -out=tfplan -var-file=environments/prod/terraform.tfvars
# Upload tfplan as a CI artifact; post the plan output as a PR comment
```

On merge to main — apply the saved plan:
```bash
terraform apply tfplan
# tfplan from the PR stage ensures what was reviewed is what gets applied
```

Never run `terraform apply` without a saved plan file in CI — it re-plans and could apply drift that wasn't reviewed.

## Output format

Provide all files in full. For each file, use a fenced code block labeled with the filename:

```hcl
# versions.tf
...
```

After the code, include:
1. A **Usage** section with the exact init/plan/apply commands including any required env vars
2. A **Security notes** section flagging any trade-offs or assumptions made (e.g. overly broad IAM that should be tightened once the app is defined)

Flag any resource that should use `prevent_destroy = true` in production and explain why.
