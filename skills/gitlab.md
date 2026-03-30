---
name: gitlab
description: Write well-structured GitLab CI/CD pipelines following stages, rules, YAML anchors, caching, security scanning, and deployment best practices
---

Analyze the current project and write a production-grade `.gitlab-ci.yml`. If one already exists, review and improve it.

## Requirements

**Inspect the project first:**
- Detect language, runtime, package manager, test framework
- Check for an existing `.gitlab-ci.yml`
- Identify deployment targets (Kubernetes, AWS, GCP, bare metal, etc.)

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

### Environments and deployments

- Use the `environment` keyword for deployment jobs to enable deployment tracking, rollbacks, and environment dashboards:
  ```yaml
  deploy:production:
    environment:
      name: production
      url: https://example.com
  ```
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
