---
name: docker
description: Write a production-grade Dockerfile following security hardening, minimal image size, and build optimization best practices
---

Analyze the current project and write a production-grade Dockerfile. Also create or update `.dockerignore`.

## Requirements

**Inspect the project first:**
- Detect language/runtime (Node.js, Python, Go, Java, etc.) and package manager
- Identify the application entrypoint and port(s)
- Check for existing Dockerfile or .dockerignore

**Dockerfile — mandatory rules:**

1. **Multi-stage build** — always use at least two stages: `builder` and `runtime`
2. **Pinned base images** — never use `:latest`; use specific version tags (e.g. `node:20.14-alpine3.20`, `python:3.12.4-slim-bookworm`)
3. **Minimal runtime image** — prefer distroless (`gcr.io/distroless/*`) or `alpine`/`slim` variants; avoid full OS images in the runtime stage
4. **Non-root user** — create a dedicated user in runtime stage and switch to it with `USER`; never run as root
5. **Layer caching** — copy dependency manifests (`package.json`, `requirements.txt`, `go.mod`, etc.) and install deps BEFORE copying source code
6. **`COPY --chown`** — always set correct ownership when copying files into the runtime stage
7. **No secrets in layers** — never hardcode secrets; use `ARG` only for non-sensitive build-time values; document env vars with `ENV` placeholders
8. **`HEALTHCHECK`** — add a meaningful `HEALTHCHECK` instruction appropriate for the app
9. **Explicit `WORKDIR`** — always set `/app` or a named workdir; never rely on root `/`
10. **Minimal `COPY`** — only copy what is needed in each stage; never `COPY . .` into the runtime stage if the build output is sufficient
11. **`CMD` vs `ENTRYPOINT`** — use `ENTRYPOINT` for the executable and `CMD` for default arguments; use exec form (JSON array), never shell form

**.dockerignore — mandatory exclusions:**
- `.git`, `.gitignore`
- `node_modules`, `__pycache__`, `*.pyc`, `.venv`, `target/`, `dist/`, `build/`
- `.env`, `.env.*`, `*.local`
- Test files and directories (`tests/`, `spec/`, `*.test.*`, `*.spec.*`)
- CI/CD config, docs, IDE config files
- The Dockerfile itself and `.dockerignore`

## Output format

Provide:
1. The complete `Dockerfile` with inline comments explaining each non-obvious decision
2. The complete `.dockerignore`
3. A short build/run reference at the end (as a code block comment) showing the exact `docker build` and `docker run` commands, including port mapping and any required env vars

## Example structure (adapt to the actual project)

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Build
FROM node:20.14-alpine3.20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

### Stage 2: Runtime
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
WORKDIR /app
# Copy only the built output and production deps
COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --from=builder --chown=nonroot:nonroot /app/node_modules ./node_modules
USER nonroot
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/nodejs/bin/node", "-e", "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"]
ENTRYPOINT ["/nodejs/bin/node"]
CMD ["dist/index.js"]
```

Flag any trade-offs made (e.g. distroless vs alpine, image size vs debuggability) and explain why each decision was made.
