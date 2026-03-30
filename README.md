# DevOps Claude Skills — AI-Powered Templates for Docker, Terraform, AWS CDK, Ansible, GitHub Actions & GitLab CI

> **Supercharge your DevOps workflow with Claude Code skills** — slash commands that generate production-grade infrastructure code, hardened Dockerfiles, and battle-tested CI/CD pipelines in seconds.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## What Are DevOps Claude Skills?

[Claude Code](https://claude.ai/code) supports **skills** — reusable prompt files you install once and invoke as slash commands (e.g. `/docker`, `/terraform`). Each skill in this repository teaches Claude exactly how to write expert-level DevOps code for a specific tool, with every best practice baked in from the start.

Instead of manually looking up whether your Dockerfile uses a non-root user, whether your Terraform variables are validated, or whether your GitHub Actions workflow pins SHA hashes — just invoke the skill and get production-ready output immediately.

**These devops Claude skills cover:**
- Security hardening out of the box
- Minimal image sizes and optimized build caching
- Least-privilege IAM, encrypted storage, and secret management
- Idempotent, lint-compliant, role-structured code
- CI/CD pipelines with proper job separation, caching, and deployment gates

---

## Available Skills

| Slash Command | Tool | What Claude Writes |
|---|---|---|
| `/docker` | Docker | Multi-stage Dockerfile + `.dockerignore` — minimal image, non-root user, pinned versions, healthcheck |
| `/terraform` | Terraform | Module structure (`main`, `variables`, `outputs`, `versions`, `backend`) — remote state, least-privilege IAM, validated variables |
| `/aws-cdk` | AWS CDK | Construct-separated stacks — encryption at rest, `grant*` IAM, CDK Aspects, env-agnostic design |
| `/ansible` | Ansible | Role-based playbooks — idempotent tasks, handlers, Vault for secrets, `block/rescue`, ansible-lint compliant |
| `/github-actions` | GitHub Actions | SHA-pinned workflows — least-privilege permissions, concurrency groups, matrix builds, dependency caching |
| `/gitlab` | GitLab CI | Staged pipelines — YAML anchors, `rules` (not `only/except`), DAG via `needs`, artifact expiry, SAST included |

---

## Installation

Skills can be installed globally (available in every project) or per-project.

### Global install (recommended)

```bash
# Clone this repository
git clone https://github.com/your-username/claude-skills-devops.git

# Install all skills globally
cp claude-skills-devops/skills/*.md ~/.claude/skills/
```

### Per-project install

```bash
# Install a single skill into your project
mkdir -p .claude/skills
cp path/to/claude-skills-devops/skills/docker.md .claude/skills/
```

### Verify installation

Open Claude Code in any project and run:

```
/help
```

You should see the installed skills listed under your available commands.

---

## Usage

Navigate to any project and invoke the relevant skill. Claude will inspect your project context and generate code tailored to your stack.

### Docker

```
/docker
```

Claude analyzes your project (language, entrypoint, ports) and generates:
- A multi-stage `Dockerfile` with distroless or alpine runtime, non-root user, pinned base image, layer-cache-optimized dependency installation, and a `HEALTHCHECK`
- A `.dockerignore` excluding dev dependencies, secrets, test files, and IDE config

### Terraform

```
/terraform
```

Claude generates a complete module with `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, and `backend.tf` — including remote state (S3 + DynamoDB), validated variables, sensitive output markers, and a mandatory tagging strategy.

### AWS CDK

```
/aws-cdk
```

Claude writes environment-agnostic CDK stacks with construct separation, explicit `RemovalPolicy`, encryption at rest for all resources, `grant*`-based IAM, CDK Aspects for cross-cutting tagging, and `CfnOutput` for cross-stack references.

### Ansible

```
/ansible
```

Claude scaffolds a full role structure with idempotent tasks, handlers for service restarts, Ansible Vault integration for secrets, `block/rescue/always` error handling, `no_log` on sensitive tasks, and ansible-lint–compliant FQCN module names.

### GitHub Actions

```
/github-actions
```

Claude generates a CI/CD workflow with SHA-pinned action versions, `permissions: {}` at workflow level with per-job grants, `concurrency` groups to cancel stale runs, dependency caching keyed by lock file hash, matrix testing, and a protected deploy job gated on `github.ref == 'refs/heads/main'`.

### GitLab CI

```
/gitlab
```

Claude writes a `.gitlab-ci.yml` with explicit stages, `rules`-based triggers (no deprecated `only/except`), YAML anchors for DRY job templates, `needs`-based DAG for parallel execution, lock-file–keyed caching, artifact expiry, environment-scoped deployment tracking, and included SAST/Secret Detection templates.

---

## Why Use These Claude Skills for DevOps?

**Without skills:** You write a Dockerfile, realize it runs as root, add a user, then later notice the base image is `:latest`, then later find the build output isn't cached properly.

**With `/docker`:** Claude generates a hardened, optimized, production-ready Dockerfile in one shot — every best practice applied from the start.

These Claude Code skills for DevOps act as an expert reviewer, code generator, and best-practices enforcer rolled into a single slash command. Whether you're writing your first Terraform module or setting up a GitLab CI pipeline for a microservices platform, each skill ensures you don't miss the details that matter in production.

---

## Contributing

Contributions are welcome. To add a new skill or improve an existing one:

1. Fork the repository
2. Create your skill file in `skills/<tool-name>.md` using the frontmatter format:
   ```markdown
   ---
   name: skill-name
   description: One-line description shown in /help
   ---

   [Prompt content...]
   ```
3. Focus the skill on **writing code**, not running commands
4. Embed concrete best practices — be specific about what Claude should and should not do
5. Open a pull request with a brief description of what the skill generates and why the best practices were chosen

---

## License

MIT — see [LICENSE](LICENSE).
