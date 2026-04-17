---
name: docker-builder
description: Generate optimized Dockerfiles and GitHub Actions docker build workflows. Covers multi-stage builds, layer caching, BuildKit, image size reduction, multi-arch builds, and post-build scanning. Scoped to build and scan — no push or deploy.
---

# Docker Builder

Expert at **Dockerfile creation and Docker image builds in GitHub Actions** — optimized for speed, security, and minimal image size. Scoped to **build and scan only** — no push, no deploy.

## When to Use This Skill

- User needs a Dockerfile for their project
- Pipeline needs to build Docker images
- User wants to optimize Docker build speed or image size
- User asks about multi-stage builds, BuildKit, or multi-arch
- Dockerfile exists but is slow or produces large images

---

## Multi-Stage Dockerfile Templates

### Go

```dockerfile
# syntax=docker/dockerfile:1

# === Build stage ===
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Cache dependency downloads (this layer changes less often)
COPY go.mod go.sum ./
RUN go mod download

# Copy source and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# === Final stage ===
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/server /server

USER nonroot:nonroot
EXPOSE 8080

ENTRYPOINT ["/server"]
```

**Key points**:
- `CGO_ENABLED=0` → static binary, no libc dependency
- `-ldflags="-s -w"` → strip debug info, smaller binary
- `distroless/static` → no shell, no package manager, minimal attack surface
- `nonroot` → never run as root

### Node.js

```dockerfile
# syntax=docker/dockerfile:1

# === Build stage ===
FROM node:22-alpine AS builder

WORKDIR /app

# Cache dependency installation
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund

# Copy source and build
COPY . .
RUN npm run build

# Prune dev dependencies
RUN npm prune --production

# === Final stage ===
FROM node:22-alpine

RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -D appuser

WORKDIR /app

# Copy only production dependencies and build output
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser
EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Python

```dockerfile
# syntax=docker/dockerfile:1

# === Build stage ===
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies
RUN pip install --no-cache-dir build

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install dependencies (cached layer)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source
COPY . .

# === Final stage ===
FROM python:3.12-slim

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder --chown=appuser:appgroup /opt/venv /opt/venv
COPY --from=builder --chown=appuser:appgroup /app .

ENV PATH="/opt/venv/bin:$PATH"
USER appuser
EXPOSE 8000

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Java (Maven)

```dockerfile
# syntax=docker/dockerfile:1

# === Build stage ===
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

# Cache Maven dependencies
COPY pom.xml .
RUN mvn dependency:go-offline -q

# Copy source and build
COPY src ./src
RUN mvn package -DskipTests -q

# === Final stage ===
FROM eclipse-temurin:21-jre-alpine

RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -D appuser

WORKDIR /app

COPY --from=builder --chown=appuser:appgroup /app/target/*.jar app.jar

USER appuser
EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Rust

```dockerfile
# syntax=docker/dockerfile:1

# === Build stage ===
FROM rust:1.77-alpine AS builder

RUN apk add --no-cache musl-dev

WORKDIR /app

# Cache dependency compilation
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Copy source and build
COPY . .
RUN touch src/main.rs && cargo build --release

# === Final stage ===
FROM alpine:3.19

RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -D appuser

COPY --from=builder --chown=appuser:appgroup /app/target/release/myapp /usr/local/bin/

USER appuser
EXPOSE 8080

ENTRYPOINT ["myapp"]
```

---

## GitHub Actions Docker Build Workflow

### Basic Build (No Push)

```yaml
# .github/workflows/docker.yml
name: Docker Build

on:
  push:
    branches: [main]
    paths:
      - 'Dockerfile'
      - 'src/**'
      - '.github/workflows/docker.yml'
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: docker-${{ github.ref }}
  cancel-in-progress: true

jobs:
  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4

      # Set up BuildKit for advanced features
      - uses: docker/setup-buildx-action@v3

      # Build with cache (no push)
      - name: Build image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false  # Build only — no push
          tags: myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # Load into local Docker for scanning
          load: true

      # Scan the built image
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: table
          severity: CRITICAL,HIGH
          exit-code: 1
```

### Multi-Architecture Build

```yaml
jobs:
  docker-build:
    name: Build Multi-Arch
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      # QEMU for cross-platform emulation
      - uses: docker/setup-qemu-action@v3

      # BuildKit with multi-platform support
      - uses: docker/setup-buildx-action@v3

      - name: Build for multiple architectures
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          platforms: linux/amd64,linux/arm64
          tags: myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## BuildKit Cache Strategies

### GitHub Actions Cache (Recommended)

```yaml
- uses: docker/build-push-action@v6
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

- `mode=max` — cache all layers, not just final stage
- Uses GitHub's built-in cache storage (10 GB limit per repo)
- No registry needed

### Inline Cache (Alternative)

```yaml
- uses: docker/build-push-action@v6
  with:
    cache-from: type=registry,ref=registry/myapp:buildcache
    cache-to: type=inline
```

### BuildKit Mount Caches in Dockerfile

For package manager caches inside the build:

```dockerfile
# Cache pip downloads across builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache Go modules across builds
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Cache Maven repository across builds
RUN --mount=type=cache,target=/root/.m2/repository \
    mvn dependency:go-offline

# Cache npm downloads across builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

---

## Layer Optimization

### Order Matters

Layers that change **less frequently** should come **first**:

```dockerfile
# ✅ GOOD — dependencies change less often than source code
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ❌ BAD — any source change invalidates npm ci cache
COPY . .
RUN npm ci
RUN npm run build
```

### Minimize Layers

Combine `RUN` commands to reduce layers:

```dockerfile
# ✅ GOOD — single layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# ❌ BAD — 3 layers, cache not cleaned in same layer
RUN apt-get update
RUN apt-get install curl
RUN rm -rf /var/lib/apt/lists/*
```

---

## .dockerignore

**Every Docker project must have a `.dockerignore`**. Without it, the entire directory (including `.git`, `node_modules`, etc.) is sent as build context.

```gitignore
# .dockerignore

# Version control
.git
.gitignore

# CI/CD
.github
.gitlab-ci.yml

# IDE
.vscode
.idea
*.swp

# Dependencies (installed in container)
node_modules
vendor
.venv
__pycache__

# Build outputs (built in container)
dist
build
target
bin

# Documentation
*.md
LICENSE
docs

# Test files
*_test.go
**/*.test.js
**/*.spec.js
tests/
test/

# Environment files
.env
.env.local
.env*.local

# Docker
Dockerfile
docker-compose*.yml
.dockerignore
```

---

## Image Size Reduction

### Base Image Selection

| Base | Size | Shell? | Use Case |
|------|------|--------|----------|
| `scratch` | 0 MB | No | Static Go binaries |
| `distroless/static` | ~2 MB | No | Static binaries with CA certs |
| `distroless/base` | ~20 MB | No | dynamically-linked binaries |
| `alpine` | ~7 MB | Yes | When you need a shell |
| `slim` variants | ~50-100 MB | Yes | When you need apt packages |
| `ubuntu/debian` | ~100-200 MB | Yes | Full OS (avoid in production) |

### Image Labels (OCI Annotations)

```yaml
# In docker/build-push-action:
- uses: docker/build-push-action@v6
  with:
    labels: |
      org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
      org.opencontainers.image.revision=${{ github.sha }}
      org.opencontainers.image.created=${{ github.event.head_commit.timestamp }}
```

Or in Dockerfile:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.description="My application"
LABEL org.opencontainers.image.licenses="MIT"
```

---

## Build Arguments and Secrets

### Build Arguments

```yaml
- uses: docker/build-push-action@v6
  with:
    build-args: |
      APP_VERSION=${{ github.sha }}
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

```dockerfile
ARG APP_VERSION=dev
ARG BUILD_DATE=unknown

LABEL app.version="${APP_VERSION}" \
      app.build-date="${BUILD_DATE}"
```

### Secrets (Never Bake Into Image)

```yaml
- uses: docker/build-push-action@v6
  with:
    secrets: |
      npm_token=${{ secrets.NPM_TOKEN }}
```

```dockerfile
# Secret is available only during this RUN, not in the final image
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    npm ci
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| `COPY . .` before installing dependencies | Every code change busts cache | Copy lockfile first, install, then copy source |
| Running as root | Security vulnerability | Add `USER nonroot` / `USER appuser` |
| No `.dockerignore` | Slow builds, secrets leaked to context | Always create `.dockerignore` |
| Using `latest` tag for base images | Non-reproducible builds | Pin versions (`node:22-alpine`, not `node:latest`) |
| Installing dev dependencies in final image | Bloated image, larger attack surface | Use multi-stage builds, prune dev deps |
| `apt-get install` without `--no-install-recommends` | Pulls in unnecessary packages | Always use `--no-install-recommends` |
| Not cleaning package manager cache | Wasted space in layer | `rm -rf /var/lib/apt/lists/*` in same `RUN` |
| Storing secrets in `ENV` or `ARG` | Visible in image layers | Use `--mount=type=secret` |
| No health check | Container orchestrator can't detect failures | Add `HEALTHCHECK` instruction |
| Using full OS base image | 500 MB+ images | Use `alpine`, `slim`, `distroless`, or `scratch` |

---

## Related Skills

When a task involves more than Docker builds, load these:

- **Caching dependencies inside Docker?** → `skill({ name: "dependency-manager" })` for cache strategies
- **Scanning built images?** → `skill({ name: "security-scanner" })` for Trivy/Grype container scanning
- **Designing the full pipeline?** → `skill({ name: "devops-pipeline-architect" })` for pipeline structure
- **Docker build failing?** → `skill({ name: "workflow-debugger" })` for diagnosing BuildKit and action issues
- **Optimizing multi-stage build speed?** → `skill({ name: "build-optimizer" })` for GHA build cache strategies

