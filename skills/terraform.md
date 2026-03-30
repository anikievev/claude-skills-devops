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
- Use workspaces or separate state files per environment

### `outputs.tf`
- Export all values that consuming modules or CI/CD pipelines will need
- Mark sensitive outputs with `sensitive = true`
- Always include `description` on every output

### IAM & Security
- Write least-privilege IAM policies — no `*` actions unless absolutely required and justified with a comment
- Never hardcode credentials, account IDs, or ARNs — use data sources and variables
- Enable encryption at rest and in transit for all supported resources
- Use `random_id` or `random_pet` for unique name suffixes where needed

### Naming conventions
- Use kebab-case for resource names: `my-service-prod`
- Prefix all resources with `${var.project}-${var.environment}-`
- Be explicit — avoid abbreviations that reduce readability

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
