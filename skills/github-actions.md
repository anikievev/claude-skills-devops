---
name: github-actions
description: Write well-structured GitHub Actions workflows following security hardening, SHA-pinned actions, least-privilege permissions, and CI/CD best practices
---

Analyze the current project and write a production-grade GitHub Actions workflow. If workflows already exist, review and improve them.

## Requirements

**Inspect the project first:**
- Detect language and runtime from lockfiles (`package-lock.json`, `go.sum`, `requirements.txt`, `pom.xml`) — never assume the stack from folder names
- Confirm lint/test/build commands actually exist in `package.json` scripts, `Makefile`, or equivalent before referencing them in the workflow
- Check for existing `.github/workflows/` files
- Identify deployment targets (cloud provider, container registry, etc.)

**File location:** `.github/workflows/ci.yml` (and separate `deploy.yml` if deployment is involved)

**Pre-generation validation checklist** — confirm each before writing the workflow:
- [ ] Lint command exists in the project (e.g. `npm run lint`, `golangci-lint`, `flake8`)
- [ ] Test command exists (e.g. `npm test`, `pytest`, `go test ./...`)
- [ ] Build command exists or is not needed (e.g. `npm run build`, `go build`, compiled apps only)
- [ ] Required secrets are identified and documented (not embedded in YAML)
- [ ] Cache key matches the actual package manager and lockfile path

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

### Deployment gates — enforce this ordering strictly
- `lint` must pass before `test`
- `test` must pass before `build`
- `build` artifact must exist before any `deploy` job starts
- Staging deploy: auto-trigger on push to `develop` or `main`
- Production deploy: always `environment` with required reviewers + `when: manual` equivalent

### Environment promotion pattern
```yaml
deploy-staging:
  needs: [build]
  if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
  environment:
    name: staging
    url: https://staging.example.com

deploy-production:
  needs: [build]
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  environment:
    name: production      # configure required reviewers in GitHub repo settings
    url: https://example.com
```

### Rollback
Every deploy job must document its rollback procedure — either as a manual workflow or as a comment:
```yaml
deploy-production:
  steps:
    - name: Deploy
      run: ./scripts/deploy.sh ${{ github.sha }}
    # Rollback: re-run this workflow with the previous passing SHA
    # or: gh workflow run rollback.yml -f sha=<previous-sha>
```

### Environment protection
- Use GitHub Environments for deployment jobs with required reviewers
- Reference environment secrets via `environment:` key — not global secrets where possible
- Use `environment.url` to surface deployment URL in the Actions UI

### Artifact handling
- Upload build artifacts with `actions/upload-artifact` and set `retention-days`
- Download only in jobs that explicitly need them via `needs`

### Docker image builds
When the project has a `Dockerfile`, add a dedicated `docker` job after tests pass. Key rules:

1. **Always use `docker/setup-buildx-action`** — enables BuildKit, multi-platform builds, and advanced caching
2. **Cache layers via GitHub Actions cache** (`cache-from`/`cache-to`) — dramatically speeds up rebuilds when only source changes
3. **Build on PR, push on merge** — `push: ${{ github.event_name == 'push' }}` prevents registry pollution from PR branches
4. **Use `docker/metadata-action`** to generate tags automatically — produces semver tags, SHA tags, and `latest` from a single source of truth
5. **Pin all Docker actions by SHA** — same rule as all other actions
6. **Scan the built image** before pushing — run `docker/scout-action` or `aquasecurity/trivy-action` against the local image

```yaml
docker:
  needs: [test]
  runs-on: ubuntu-latest
  timeout-minutes: 20
  permissions:
    contents: read
    packages: write      # required to push to GHCR

  steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

    # Generate image tags and labels from git metadata
    - uses: docker/metadata-action@369eb591f429131d6889c46b94e711f089e6ca96  # v5.6.1
      id: meta
      with:
        images: ghcr.io/${{ github.repository }}
        tags: |
          type=sha,prefix=sha-
          type=ref,event=branch
          type=semver,pattern={{version}}
          type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

    # Enable BuildKit and multi-platform support
    - uses: docker/setup-buildx-action@f7ce87c1d6bead3e36075b2ce75da1f6cc28aaca  # v3.9.0

    # Log in to registry — only needed when pushing (merge to main)
    - uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567  # v3.3.0
      if: github.event_name == 'push'
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}   # built-in, no extra secret needed for GHCR

    # Build (and push only on merge)
    - uses: docker/build-push-action@0adf9959216b96bec444f325f1e493d4aa344a20  # v6.14.0
      id: build
      with:
        context: .
        push: ${{ github.event_name == 'push' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        # Layer cache: pull existing layers from registry, push new ones after build
        cache-from: type=gha
        cache-to: type=gha,mode=max

    # Scan the built image for CVEs before it lands in the registry
    - uses: aquasecurity/trivy-action@915b19bbe73b92a6cf82a1bc12b087c9a19a5fe2  # v0.29.0
      with:
        image-ref: ${{ steps.build.outputs.imageid }}
        format: table
        exit-code: 1           # fail the job if CRITICAL vulnerabilities are found
        severity: CRITICAL,HIGH
```

**Registry options:**
- **GHCR** (shown above) — free for public repos, uses `GITHUB_TOKEN`, no extra secret
- **Docker Hub** — use `docker/login-action` with `${{ secrets.DOCKERHUB_USERNAME }}` and `${{ secrets.DOCKERHUB_TOKEN }}`
- **AWS ECR** — use `aws-actions/amazon-ecr-login` after configuring OIDC role (no long-lived AWS keys)

**Multi-platform builds** (arm64 + amd64) — add QEMU setup before buildx:
```yaml
    - uses: docker/setup-qemu-action@29109295f81e9208d7d86ff9f1f6a21f4f0bcd9f  # v3.3.0
    - uses: docker/setup-buildx-action@f7ce87c1d6bead3e36075b2ce75da1f6cc28aaca  # v3.9.0
    # Then add to build-push-action:
    # platforms: linux/amd64,linux/arm64
```

### Common pitfalls — never do these
- **Copying a Node.js workflow into a Python or Go repo** — cache paths, setup actions, and install commands are all different; always regenerate from the detected stack
- **Enabling deploy jobs before tests are stable** — a flaky test suite will deploy broken code; fix flakiness first
- **Hardcoded secrets in YAML** — any value in the workflow file is readable by anyone with repo access; use `${{ secrets.NAME }}`
- **Running the full matrix on every branch** — matrix builds are expensive; limit to PRs targeting `main` and pushes to `main`
- **Missing branch protection** — deploy jobs without branch protection rules can be triggered from any fork PR; always gate on `github.event_name == 'push'` and a protected branch

## Output format

Provide the full workflow file(s) in fenced code blocks:

```yaml
# .github/workflows/ci.yml
...
```

After the code, include:
1. A **SHA pinning note**: list the actions used and their pinned SHAs with version comments
2. A **Permissions map**: table showing which job gets which permissions and why
3. A **Required secrets** section: every `${{ secrets.X }}` reference listed with its purpose
4. A **Security notes** section flagging any trade-offs (e.g. `contents: write` needed for release commits)

---

## Framework examples

Use the example matching the detected stack as a starting point. Always replace tag-based action refs with SHA-pinned refs before delivering.

---

### Node.js (npm / yarn / pnpm)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
    strategy:
      matrix:
        node: ['18', '20', '22']
      fail-fast: false
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08  # v4.6.0
        with:
          name: dist-${{ github.sha }}
          path: dist/
          retention-days: 7
```

---

### Python (FastAPI / Django / Flask)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-python@0b93645e9fea7318ecaed2b359559ac225c90a2b  # v5.3.0
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install --no-cache-dir -r requirements-dev.txt
      - run: ruff check .          # or: flake8 . / pylint

  test:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
    strategy:
      matrix:
        python: ['3.11', '3.12']
      fail-fast: false
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-python@0b93645e9fea7318ecaed2b359559ac225c90a2b  # v5.3.0
        with:
          python-version: ${{ matrix.python }}
          cache: 'pip'
      - run: pip install --no-cache-dir -r requirements-dev.txt
      - run: pytest --tb=short -q
```

---

### Go

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-go@f111f3307d8850f501ac008e886eec1fd1932a34  # v5.3.0
        with:
          go-version-file: go.mod
          cache: true
      - uses: golangci/golangci-lint-action@2226d7cb06a077cd73e56eedd38eecad18e5d837  # v6.5.0
        with:
          version: latest

  test:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-go@f111f3307d8850f501ac008e886eec1fd1932a34  # v5.3.0
        with:
          go-version-file: go.mod
          cache: true
      - run: go test -race -coverprofile=coverage.out ./...
      - run: go build ./...

  build:
    needs: test
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-go@f111f3307d8850f501ac008e886eec1fd1932a34  # v5.3.0
        with:
          go-version-file: go.mod
          cache: true
      - run: CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o bin/server ./cmd/server
      - uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08  # v4.6.0
        with:
          name: binary-${{ github.sha }}
          path: bin/
          retention-days: 7
```

---

### Java / Spring Boot (Maven)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: actions/setup-java@3a4f6e1af504cf6a31b2d8c4b09f55b5e8c8e88f  # v4.7.0
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      - run: ./mvnw verify -B -q    # runs compile + test + package; -B = non-interactive
      - uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08  # v4.6.0
        with:
          name: jar-${{ github.sha }}
          path: target/*.jar
          retention-days: 7
```

> **Gradle alternative:** Replace `actions/setup-java` cache value with `gradle` and the run step with `./gradlew build -x test` (lint) + `./gradlew test` (test) + `./gradlew bootJar` (build).

---

### Docker image build + push (any stack)

Add this job after your language-specific `test` job. Works for any framework that has a `Dockerfile`.

```yaml
# .github/workflows/ci.yml  (append to existing jobs)

  docker:
    needs: [test]           # replace with your actual test job name
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
      packages: write       # push to GHCR

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - uses: docker/metadata-action@369eb591f429131d6889c46b94e711f089e6ca96  # v5.6.1
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - uses: docker/setup-buildx-action@f7ce87c1d6bead3e36075b2ce75da1f6cc28aaca  # v3.9.0

      - uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567  # v3.3.0
        if: github.event_name == 'push'
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@0adf9959216b96bec444f325f1e493d4aa344a20  # v6.14.0
        id: build
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - uses: aquasecurity/trivy-action@915b19bbe73b92a6cf82a1bc12b087c9a19a5fe2  # v0.29.0
        with:
          image-ref: ${{ steps.build.outputs.imageid }}
          format: table
          exit-code: 1
          severity: CRITICAL,HIGH
```
