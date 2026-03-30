---
name: gitlab
description: Write well-structured GitLab CI/CD pipelines following stages, rules, YAML anchors, caching, security scanning, and deployment best practices
---

Analyze the current project and write a production-grade `.gitlab-ci.yml`. If one already exists, review and improve it.

## Requirements

**Inspect the project first:**
- Detect language and runtime from lockfiles (`package-lock.json`, `go.sum`, `requirements.txt`, `pom.xml`) — never assume the stack from folder names
- Confirm lint/test/build commands actually exist in `package.json` scripts, `Makefile`, or equivalent before referencing them in the pipeline
- Check for an existing `.gitlab-ci.yml`
- Identify deployment targets (Kubernetes, AWS, GCP, bare metal, etc.)

**Pre-generation validation checklist** — confirm each before writing the pipeline:
- [ ] Lint command exists in the project
- [ ] Test command exists
- [ ] Build command exists or is not needed
- [ ] Required CI/CD variables are identified and documented at the top of the file
- [ ] Cache key matches the actual lockfile path for this project

**Mandatory rules:**

### Pipeline structure

Always define explicit `stages` — never rely on default ordering:

```yaml
stages:
  - build
  - test
  - security
  - package
  - deploy
```

### Rules over only/except

Never use deprecated `only`/`except`. Always use `rules`:

```yaml
# Good
rules:
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    when: on_success
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    when: on_success
  - when: never
```

### YAML anchors for DRY config

Use anchors (`&`) and merge keys (`<<: *`) to avoid duplication:

```yaml
.default_rules: &default_rules
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

lint:
  <<: *default_rules
  stage: test
  script: npm run lint
```

### Caching

Cache based on lock file hash — never use a fixed key:

```yaml
cache:
  key:
    files:
      - package-lock.json   # or requirements.txt, go.sum, etc.
  paths:
    - node_modules/
  policy: pull-push  # pull in test jobs; push only in build
```

For test/downstream jobs, use `policy: pull` to avoid unnecessary uploads.

### Artifacts

- Set `expire_in` on all artifacts — never keep them indefinitely
- Use `expose_as` to surface artifacts in merge request UI
- Upload test reports as JUnit artifacts for native MR test reporting:
  ```yaml
  artifacts:
    when: always
    expire_in: 7 days
    reports:
      junit: report.xml
    paths:
      - dist/
  ```

### DAG with `needs`

Use `needs` to create a directed acyclic graph — don't wait for an entire stage if only one upstream job is required:

```yaml
test:unit:
  stage: test
  needs: [build]
  ...

test:e2e:
  stage: test
  needs: [build]
  ...

deploy:staging:
  stage: deploy
  needs: [test:unit, test:e2e]
  ...
```

### Secrets and variables

- Store secrets as **Protected** and **Masked** CI/CD variables — never hardcode in `.gitlab-ci.yml`
- Use **Environment-scoped** variables for environment-specific secrets
- Reference variables as `$VARIABLE_NAME` — document required variables in a comment block at the top of the file
- Never print secrets with `echo` — use masked variables and check masking works for the secret format

### Deployment gates — enforce this ordering strictly
- `lint` must pass before `test` (use `needs`)
- `test` must pass before `build`
- Build artifact must exist before any deploy job starts
- Staging: auto-deploy on `$CI_COMMIT_BRANCH == "develop"` or `$CI_DEFAULT_BRANCH`
- Production: always `when: manual` with a protected environment

### Environment promotion pattern
```yaml
deploy:staging:
  stage: deploy
  needs: [build]
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
      when: on_success
    - when: never

deploy:production:
  stage: deploy
  needs: [build]
  when: manual                    # requires human approval
  allow_failure: false
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - when: never
```

### Rollback
Every deploy job must document its rollback procedure:
```yaml
deploy:production:
  script:
    - ./scripts/deploy.sh $CI_COMMIT_SHA
  # Rollback: re-run the previous successful pipeline for the last known-good SHA
  # or: trigger a manual deploy:production job on that pipeline
  environment:
    name: production
    on_stop: rollback:production   # define a mirror job that deploys the previous version

rollback:production:
  stage: deploy
  when: manual
  environment:
    name: production
    action: stop
  script:
    - ./scripts/deploy.sh $PREVIOUS_STABLE_SHA
```

### Environments and deployments

- Use the `environment` keyword for deployment jobs to enable deployment tracking, rollbacks, and environment dashboards
- Add `when: manual` and `allow_failure: false` for production deploys

### Security scanning

Include GitLab's built-in security template for SAST at minimum:

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
```

Customize scanner variables if needed; do not disable scanners without a documented reason.

### Runner configuration

- Specify `tags` to target the correct runner for each job
- Set `interruptible: true` for non-deploy jobs to allow pipeline cancellation
- Set a `timeout` on long-running jobs

### Common pitfalls — never do these
- **Copying a Node.js pipeline into a Python or Go repo** — images, cache keys, and commands are all different; always regenerate from the detected stack
- **Using `only`/`except`** — deprecated; use `rules` exclusively
- **Fixed cache keys** (`key: "node-cache"`) — cache is never invalidated when dependencies change; always key on the lockfile hash
- **Installing dependencies twice** (once in lint, once in test) without caching — use `policy: pull-push` in the first job, `policy: pull` in subsequent ones
- **Enabling deploy jobs before tests are stable** — a flaky test suite will promote broken artifacts to staging or production
- **Hardcoded secrets in YAML** — any value in `.gitlab-ci.yml` is readable by anyone with repo access; use Protected + Masked CI/CD variables
- **`allow_failure: true` on security scan jobs** — silently ignores vulnerabilities; only use on known-flaky jobs with a tracking issue

## Output format

Provide the complete `.gitlab-ci.yml` in a fenced code block:

```yaml
# .gitlab-ci.yml
...
```

After the code, include:
1. A **Required variables** section: table of all CI/CD variables the pipeline expects, their purpose, and whether they should be Protected/Masked
2. A **Pipeline graph** section: ASCII or text description of the job DAG
3. A **Security notes** section flagging any jobs with elevated permissions, manual gates, or disabled security scanners

---

## Framework examples

Use the example matching the detected stack as a starting point.

---

### Node.js (npm)

```yaml
# .gitlab-ci.yml

# Required CI/CD variables: none for CI; add DEPLOY_TOKEN for deploy stage

stages:
  - lint
  - test
  - build
  - deploy

.node_cache: &node_cache
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/

.default_rules: &default_rules
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - when: never

lint:
  <<: *default_rules
  stage: lint
  image: node:20-alpine
  cache:
    <<: *node_cache
    policy: pull-push
  script:
    - npm ci
    - npm run lint
  interruptible: true

test:
  <<: *default_rules
  stage: test
  image: node:20-alpine
  needs: [lint]
  cache:
    <<: *node_cache
    policy: pull
  script:
    - npm ci
    - npm test
  artifacts:
    when: always
    expire_in: 7 days
    reports:
      junit: junit.xml
  interruptible: true

build:
  <<: *default_rules
  stage: build
  image: node:20-alpine
  needs: [test]
  cache:
    <<: *node_cache
    policy: pull
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 day
  interruptible: true
```

---

### Python (FastAPI / Django)

```yaml
# .gitlab-ci.yml

stages:
  - lint
  - test
  - build
  - deploy

.python_cache: &python_cache
  cache:
    key:
      files:
        - requirements.txt
    paths:
      - .venv/

.default_rules: &default_rules
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - when: never

lint:
  <<: *default_rules
  stage: lint
  image: python:3.12-slim
  cache:
    <<: *python_cache
    policy: pull-push
  script:
    - python -m venv .venv
    - source .venv/bin/activate
    - pip install --no-cache-dir -r requirements.txt
    - ruff check .      # or: flake8 / pylint
  interruptible: true

test:
  <<: *default_rules
  stage: test
  image: python:3.12-slim
  needs: [lint]
  cache:
    <<: *python_cache
    policy: pull
  script:
    - source .venv/bin/activate
    - pytest --tb=short -q --junitxml=report.xml
  artifacts:
    when: always
    expire_in: 7 days
    reports:
      junit: report.xml
  interruptible: true
```

---

### Go

```yaml
# .gitlab-ci.yml

stages:
  - lint
  - test
  - build
  - deploy

.go_cache: &go_cache
  cache:
    key:
      files:
        - go.sum
    paths:
      - .go-cache/

variables:
  GOPATH: $CI_PROJECT_DIR/.go-cache
  CGO_ENABLED: "0"

.default_rules: &default_rules
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - when: never

lint:
  <<: *default_rules
  stage: lint
  image: golangci/golangci-lint:v1.62-alpine
  cache:
    <<: *go_cache
    policy: pull-push
  script:
    - golangci-lint run ./...
  interruptible: true

test:
  <<: *default_rules
  stage: test
  image: golang:1.22-alpine
  needs: [lint]
  cache:
    <<: *go_cache
    policy: pull
  script:
    - go test -race -coverprofile=coverage.out ./...
  artifacts:
    when: always
    expire_in: 7 days
    paths:
      - coverage.out
  interruptible: true

build:
  <<: *default_rules
  stage: build
  image: golang:1.22-alpine
  needs: [test]
  cache:
    <<: *go_cache
    policy: pull
  script:
    - go build -ldflags="-s -w" -o bin/server ./cmd/server
  artifacts:
    paths:
      - bin/
    expire_in: 1 day
  interruptible: true
```

---

### Java / Spring Boot (Maven)

```yaml
# .gitlab-ci.yml

stages:
  - test
  - build
  - deploy

.maven_cache: &maven_cache
  cache:
    key:
      files:
        - pom.xml
    paths:
      - .m2/repository/

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository -B"

.default_rules: &default_rules
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - when: never

test:
  <<: *default_rules
  stage: test
  image: eclipse-temurin:21-jdk-alpine
  cache:
    <<: *maven_cache
    policy: pull-push
  script:
    - ./mvnw verify $MAVEN_OPTS
  artifacts:
    when: always
    expire_in: 7 days
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
  interruptible: true

build:
  <<: *default_rules
  stage: build
  image: eclipse-temurin:21-jdk-alpine
  needs: [test]
  cache:
    <<: *maven_cache
    policy: pull
  script:
    - ./mvnw package -DskipTests $MAVEN_OPTS
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 day
  interruptible: true
```
