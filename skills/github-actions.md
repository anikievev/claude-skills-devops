---
name: github-actions
description: Write well-structured GitHub Actions workflows following security hardening, SHA-pinned actions, least-privilege permissions, and CI/CD best practices
---

Analyze the current project and write a production-grade GitHub Actions workflow. If workflows already exist, review and improve them.

## Requirements

**Inspect the project first:**
- Detect language, runtime, package manager, test framework
- Check for existing `.github/workflows/` files
- Identify deployment targets (cloud provider, container registry, etc.)

**File location:** `.github/workflows/ci.yml` (and separate `deploy.yml` if deployment is involved)

**Mandatory rules:**

### Security — top priority

1. **Pin all actions by SHA commit hash**, never by mutable tag:
   ```yaml
   # Bad
   uses: actions/checkout@v4
   # Good
   uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
   ```

2. **Least-privilege `permissions`** — always declare at both workflow and job level; grant only what each job needs:
   ```yaml
   permissions: {}  # deny all at workflow level
   jobs:
     build:
       permissions:
         contents: read
   ```

3. **Never log secrets** — do not use `echo $SECRET`; ensure no step prints env vars that contain secrets

4. **Validate inputs** — if the workflow uses `workflow_dispatch` with inputs, validate them before use

5. **`CODEOWNERS` awareness** — note if branch protection rules should require review for workflow changes

### Workflow structure

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Cancel in-flight runs for the same PR/branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: {}  # Deny all by default; grant per job

jobs:
  lint:
    ...
  test:
    needs: lint
    ...
  build:
    needs: test
    ...
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    ...
```

### Job design
- Separate jobs for lint, test, build, and deploy — never combine them
- Use `needs` to enforce ordering; run independent jobs in parallel
- Set `timeout-minutes` on every job to prevent runaway builds
- Use `continue-on-error: false` (default) — never suppress failures silently

### Caching
- Always cache dependencies using `actions/cache` with a key based on the lock file hash:
  ```yaml
  - uses: actions/cache@d4323d4df104b026a6aa633fdb11d772146be0bf  # v4.2.2
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-
  ```

### Matrix builds
- Use matrix strategy for multi-version testing where relevant:
  ```yaml
  strategy:
    matrix:
      node: [18, 20, 22]
    fail-fast: false
  ```

### Environment protection
- Use GitHub Environments for deployment jobs with required reviewers
- Reference environment secrets via `environment:` key — not global secrets where possible
- Use `environment.url` to surface deployment URL in the Actions UI

### Artifact handling
- Upload build artifacts with `actions/upload-artifact` and set `retention-days`
- Download only in jobs that explicitly need them via `needs`

## Output format

Provide the full workflow file(s) in fenced code blocks:

```yaml
# .github/workflows/ci.yml
...
```

After the code, include:
1. A **SHA pinning note**: list the actions used and their pinned SHAs with version comments
2. A **Permissions map**: table showing which job gets which permissions and why
3. A **Security notes** section flagging any trade-offs (e.g. `contents: write` needed for release commits)
