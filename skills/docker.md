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
5. **Layer caching** — copy dependency manifests (`package.json`, `requirements.txt`, `go.mod`, etc.) and install deps BEFORE copying source code; order instructions from least-changed to most-changed
6. **Chain `RUN` commands** — combine related commands into a single `RUN` with `&&`; never split `apt-get update` and `apt-get install` into separate `RUN` steps; always clean package manager caches in the same layer (e.g. `rm -rf /var/lib/apt/lists/*`, `apk add --no-cache`)
7. **`COPY` over `ADD`** — always use `COPY` for local files; only use `ADD` when you explicitly need archive auto-extraction; never use `ADD` to fetch URLs
8. **`COPY --chown`** — always set correct ownership when copying files into the runtime stage
9. **No secrets in layers** — never hardcode secrets in `ENV` or `ARG`; values set via `ENV` are stored in image metadata and visible to anyone via `docker history`; inject secrets at runtime via environment variables, mounted volumes, or Docker secrets
10. **`HEALTHCHECK`** — add a meaningful `HEALTHCHECK` instruction appropriate for the app
11. **Explicit `WORKDIR`** — always set `/app` or a named workdir; never rely on root `/`
12. **Minimal `COPY`** — only copy what is needed in each stage; never `COPY . .` into the runtime stage if the build output is sufficient
13. **`CMD` vs `ENTRYPOINT`** — use `ENTRYPOINT` for the executable and `CMD` for default arguments; use exec form (JSON array), never shell form
14. **Build idempotency** — `RUN` steps must be pure: compile, transform, copy — never mutate external state (no git commits, no database calls, no external POST requests) inside a build step
15. **Vulnerability scanning** — after generating the Dockerfile, recommend running `docker scout cves <image>` or `trivy image <image>` against the built image before pushing to a registry

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
2. The complete `.dockerignore` (use the framework-specific exclusions below)
3. A short build/run reference at the end showing the exact `docker build` and `docker run` commands, including port mapping and required env vars

Flag any trade-offs made (e.g. distroless vs alpine, image size vs debuggability) and explain why each decision was made.

---

## Framework examples

Use the example matching the detected framework as a starting point. Adapt image versions to what is current and appropriate for the project.

---

### Node.js (Express / NestJS / Next.js)

Three-stage pattern: all deps for building → production deps only → distroless runtime.
Never install only `--omit=dev` in the build stage — devDependencies include the compiler/bundler.

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Build (needs devDependencies for tsc / webpack / etc.)
FROM node:20.14-alpine3.20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

### Stage 2: Production deps only
FROM node:20.14-alpine3.20 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

### Stage 3: Minimal runtime
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
WORKDIR /app
COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --from=deps    --chown=nonroot:nonroot /app/node_modules ./node_modules
USER nonroot
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/nodejs/bin/node", "-e", "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"]
ENTRYPOINT ["/nodejs/bin/node"]
CMD ["dist/index.js"]
```

**.dockerignore additions:** `node_modules/`, `dist/`, `build/`, `.next/`, `coverage/`

---

### FastAPI (Python / uvicorn)

Use a virtualenv in the builder so it can be copied as a single directory — cleaner than `--prefix` installs.

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Install dependencies into a virtualenv
FROM python:3.12.4-slim-bookworm AS builder
WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt ./
# --no-cache-dir keeps the layer small; pip is already upgraded in the base image
RUN pip install --no-cache-dir -r requirements.txt

### Stage 2: Runtime (slim, non-root)
FROM python:3.12.4-slim-bookworm AS runtime
# Create a non-root user
RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser
WORKDIR /app
# Copy the virtualenv from builder — no pip, no compiler, no build tools in runtime
COPY --from=builder --chown=appuser:appuser /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
# Copy only application source (not tests, docs, etc.)
COPY --chown=appuser:appuser ./app ./app
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
ENTRYPOINT ["uvicorn"]
CMD ["app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

**.dockerignore additions:** `__pycache__/`, `*.pyc`, `*.pyo`, `.venv/`, `.pytest_cache/`, `tests/`, `htmlcov/`

> **Trade-off:** `python:slim` is used instead of distroless/python because FastAPI's uvicorn worker occasionally needs system libraries (e.g. `libpq` for asyncpg). If the app has no native dependencies, distroless/python is a smaller alternative.

---

### Laravel (PHP / php-fpm)

Use the official `composer` image to install vendor dependencies, then copy into a php-fpm runtime. Never bundle `composer` itself in the runtime image.

**Important:** Do not run `php artisan config:cache` in the Dockerfile — it bakes runtime env vars (DB host, APP_KEY, etc.) into the image. Instead, run it via an entrypoint script at container startup.

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Composer dependencies
FROM composer:2.7 AS composer
WORKDIR /app
# Copy manifests first for layer caching
COPY composer.json composer.lock ./
# --no-scripts: defer post-install scripts that may need a full app environment
# --no-autoloader: regenerate autoloader in next step after source is present
RUN composer install \
      --no-dev \
      --no-scripts \
      --no-autoloader \
      --prefer-dist
COPY . .
# Generate optimized autoloader with full source available
RUN composer dump-autoload --optimize --no-dev

### Stage 2: Runtime (php-fpm, non-root)
FROM php:8.3-fpm-alpine3.19 AS runtime
# Install only the PHP extensions the app needs; chain && to keep one layer
RUN apk add --no-cache \
      libpng-dev \
      libjpeg-turbo-dev \
      libzip-dev \
    && docker-php-ext-install -j$(nproc) \
      pdo_mysql \
      zip \
      opcache \
    && rm -rf /tmp/pear
# OPcache tuning for production
COPY docker/opcache.ini /usr/local/etc/php/conf.d/opcache.ini
WORKDIR /var/www/html
COPY --from=composer --chown=www-data:www-data /app ./
# Entrypoint script runs artisan config:cache / route:cache at startup
COPY --chown=www-data:www-data docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
# Writable directories Laravel needs
RUN chown -R www-data:www-data storage bootstrap/cache
USER www-data
EXPOSE 9000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD php-fpm -t
ENTRYPOINT ["/entrypoint.sh"]
CMD ["php-fpm"]
```

**.dockerignore additions:** `vendor/`, `node_modules/`, `storage/logs/`, `storage/framework/cache/`, `.env`, `tests/`

> **Note:** This image runs php-fpm only. Pair it with a separate nginx container (via Docker Compose or Kubernetes sidecar) to serve static assets and proxy PHP requests.

---

### Spring Boot (Java / Maven)

Use Spring Boot's **layered JAR** feature to split the fat JAR into dependency layers. Dependencies (rarely changing) get their own Docker layer; application code (frequently changing) gets a separate layer. Rebuilds after code-only changes are fast because the dependency layer is cached.

Use `eclipse-temurin` JRE (not JDK) in the runtime stage — the JDK includes compilers and tools not needed at runtime.

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Build fat JAR with Maven wrapper
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
# Cache Maven dependencies — only re-downloaded when pom.xml changes
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -q
# Build — skip tests (tests run in CI, not in Docker build)
COPY src ./src
RUN ./mvnw package -DskipTests -q

### Stage 2: Extract layered JAR
# layertools splits the fat JAR into: dependencies / spring-boot-loader / snapshot-dependencies / application
FROM eclipse-temurin:21-jdk-alpine AS extractor
WORKDIR /extracted
COPY --from=builder /app/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

### Stage 3: Runtime (JRE only, non-root)
FROM eclipse-temurin:21-jre-alpine AS runtime
RUN addgroup -S spring && adduser -S spring -G spring
WORKDIR /app
# Copy layers in order of change frequency: dependencies first, application last
# Each COPY is a separate layer — unchanged layers are cache hits on rebuild
COPY --from=extractor --chown=spring:spring /extracted/dependencies ./
COPY --from=extractor --chown=spring:spring /extracted/spring-boot-loader ./
COPY --from=extractor --chown=spring:spring /extracted/snapshot-dependencies ./
COPY --from=extractor --chown=spring:spring /extracted/application ./
USER spring
EXPOSE 8080
# Requires spring-boot-starter-actuator on the classpath
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java"]
# UseContainerSupport: respect cgroup memory limits (critical in Kubernetes)
# MaxRAMPercentage: use up to 75% of container memory for heap
CMD ["-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "org.springframework.boot.loader.launch.JarLauncher"]
```

**.dockerignore additions:** `target/`, `.mvn/wrapper/maven-wrapper.jar` (if not using wrapper jar), `**/*.class`

> **Gradle alternative:** Replace the Maven stage with `COPY build.gradle settings.gradle gradlew ./` + `COPY gradle gradle` + `RUN ./gradlew dependencies -q` + `RUN ./gradlew bootJar -x test -q`. The layered extraction stage is identical.

---

### Go (Gin / Echo / net/http)

Go compiles to a single static binary. The runtime image can be `FROM scratch` (zero OS) or `gcr.io/distroless/static-debian12` (adds CA certs and timezone data without a shell). This produces the smallest possible images — typically 5–20 MB.

Use `CGO_ENABLED=0` and `GOOS=linux` to guarantee a fully static binary that runs in a scratch image.

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Build static binary
FROM golang:1.22-alpine3.20 AS builder
WORKDIR /app
# Download modules first — cached until go.mod / go.sum change
COPY go.mod go.sum ./
RUN go mod download
# Build with CGO disabled for a fully static binary
# -ldflags="-s -w" strips debug symbols → smaller binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app/server ./cmd/server

### Stage 2: Minimal runtime
# distroless/static: includes CA certs + timezone data, no shell, no package manager
# Use "FROM scratch" instead only if the app never makes TLS calls or uses time zones
FROM gcr.io/distroless/static-debian12 AS runtime
# Copy the binary and nothing else
COPY --from=builder --chown=nonroot:nonroot /app/server /server
USER nonroot
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD ["/server", "-healthcheck"]
ENTRYPOINT ["/server"]
```

**.dockerignore additions:** `vendor/` (if not vendoring), `**/*_test.go`, `testdata/`

> **Trade-off:** `distroless/static` is preferred over `scratch` because it includes CA certificates (required for outbound HTTPS calls) and `/etc/passwd` (required for the `nonroot` user). Use `scratch` only for apps with zero external TLS calls and no user switching.
>
> **GOARCH:** Set `GOARCH=amd64` explicitly or use `--platform` on `docker build` for cross-compilation to arm64 (e.g. for Apple Silicon CI runners building for x86 production).

---

### Django (Python / Gunicorn)

Django differs from FastAPI in three key ways that affect the Dockerfile:
1. **Static files** — `manage.py collectstatic` must run at build time to gather assets into `STATIC_ROOT`
2. **Process manager** — use Gunicorn (WSGI), not uvicorn alone (though `uvicorn[standard]` with Gunicorn workers is valid for async views)
3. **Migrations** — never run `manage.py migrate` in the Dockerfile; run it via an entrypoint script at startup so it targets the live database

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Build virtualenv + collect static files
FROM python:3.12.4-slim-bookworm AS builder
WORKDIR /app
# Install build-time OS deps (e.g. libpq-dev for psycopg2)
RUN apt-get update && apt-get install -y --no-install-recommends \
      libpq-dev \
      gcc \
    && rm -rf /var/lib/apt/lists/*
# Virtualenv in a known path so it can be copied cleanly
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
# Copy source to run collectstatic
COPY . .
# DJANGO_SETTINGS_MODULE must point to a settings file that doesn't require a DB connection
# Use a build-time settings module or set STATIC_ROOT and ALLOWED_HOSTS as ARGs
ARG DJANGO_SETTINGS_MODULE=config.settings.build
RUN python manage.py collectstatic --noinput

### Stage 2: Runtime (slim, non-root)
FROM python:3.12.4-slim-bookworm AS runtime
# Runtime OS deps only (libpq for psycopg2 at runtime; no gcc needed)
RUN apt-get update && apt-get install -y --no-install-recommends \
      libpq5 \
    && rm -rf /var/lib/apt/lists/*
RUN groupadd -r django && useradd -r -g django -d /app django
WORKDIR /app
COPY --from=builder --chown=django:django /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
# Copy application source and collected static files
COPY --from=builder --chown=django:django /app /app
# Entrypoint script runs: migrate, then exec gunicorn
COPY --chown=django:django docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
USER django
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')"
ENTRYPOINT ["/entrypoint.sh"]
# Gunicorn: 2*CPU+1 workers is the standard formula; tune via env var at runtime
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3", "--timeout", "60"]
```

**.dockerignore additions:** `__pycache__/`, `*.pyc`, `.venv/`, `staticfiles/`, `media/`, `.pytest_cache/`, `tests/`

> **Static files:** Serve `STATIC_ROOT` via nginx (separate container) or a CDN — never through Gunicorn in production. Mount `MEDIA_ROOT` as a persistent volume.
>
> **Migrations:** The entrypoint script should run `python manage.py migrate --noinput` before starting Gunicorn. In Kubernetes, use an init container for migrations instead.

---

### ASP.NET Core (.NET / Kestrel)

Use `dotnet publish` with `--self-contained false` (framework-dependent) so the runtime image can be the slim `mcr.microsoft.com/dotnet/aspnet` base instead of bundling the entire .NET runtime inside the binary.

The `mcr.microsoft.com/dotnet/sdk` image is used only in the build stage — it is large (~800 MB) and must never appear in the runtime stage.

```dockerfile
# syntax=docker/dockerfile:1

### Stage 1: Restore dependencies (separate layer for caching)
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS restore
WORKDIR /src
# Copy only the project file(s) first — restores are cached until .csproj changes
COPY ["src/MyApp/MyApp.csproj", "src/MyApp/"]
RUN dotnet restore "src/MyApp/MyApp.csproj"

### Stage 2: Build and publish
FROM restore AS builder
# Copy source and publish a Release build
COPY src/ ./src/
WORKDIR /src/src/MyApp
RUN dotnet publish "MyApp.csproj" \
      --configuration Release \
      --no-restore \
      --output /app/publish \
      /p:UseAppHost=false

### Stage 3: Runtime (aspnet runtime only, non-root)
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
# Create a non-root user
RUN addgroup -S dotnet && adduser -S dotnet -G dotnet
WORKDIR /app
COPY --from=builder --chown=dotnet:dotnet /app/publish ./
USER dotnet
EXPOSE 8080
# ASP.NET Core reads ASPNETCORE_URLS to bind; Kestrel defaults to 8080 in .NET 8+
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget -q --spider http://localhost:8080/health || exit 1
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**.dockerignore additions:** `**/bin/`, `**/obj/`, `**/*.user`, `.vs/`, `TestResults/`

> **Self-contained vs framework-dependent:** `--self-contained true` bundles the .NET runtime into the publish output, allowing `FROM scratch` or a distroless base, but increases image size by ~60 MB. Prefer framework-dependent (default) unless you need a fully portable single binary.
>
> **Alpine vs Debian:** `aspnet:8.0-alpine` is smaller but uses musl libc. If the app or its NuGet dependencies rely on glibc (e.g. native SQLite, some cryptography packages), use `aspnet:8.0-bookworm-slim` instead.
