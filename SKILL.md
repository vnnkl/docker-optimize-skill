---
name: docker-optimize
description: "Audit and optimize Dockerfiles for size, security, build speed, and best practices. Triggers on: optimize dockerfile, audit docker, fix dockerfile, docker best practices."
---

# Docker Optimization

Audit Dockerfiles for common mistakes and apply battle-tested fixes. Based on patterns from hundreds of real-world Docker deployments.

---

## The Job

1. Read the Dockerfile(s) in the project
2. Audit against the 10 critical mistakes checklist
3. Generate a prioritized findings report
4. Apply fixes (with user confirmation)

---

## Audit Checklist

Run through each check. Score: 🔴 Critical | 🟡 Warning | 🟢 Good

### 1. Base Image Tags
```dockerfile
# 🔴 BAD: Moving target
FROM node:latest

# 🟢 GOOD: Pinned version
FROM node:20.11.1-alpine3.19
```
**Check**: Any `FROM` with `:latest` or no tag → 🔴 Critical

### 2. Image Size
```dockerfile
# 🔴 BAD: Full OS (1.8GB+)
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nodejs npm

# 🟢 GOOD: Minimal base (180MB)
FROM node:20.11-alpine
```
**Check**: Using `ubuntu`, `debian` (non-slim), full `node`/`python` → 🟡 Warning
**Recommend**: alpine or slim variants

### 3. Layer Cache Optimization
```dockerfile
# 🔴 BAD: Cache busted on every code change
COPY . .
RUN npm install

# 🟢 GOOD: Dependencies cached separately
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```
**Check**: `COPY . .` before dependency install → 🔴 Critical

### 4. Non-Root User
```dockerfile
# 🔴 BAD: Running as root
CMD ["node", "server.js"]

# 🟢 GOOD: Dedicated user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
CMD ["node", "server.js"]
```
**Check**: No `USER` instruction before final `CMD` → 🟡 Warning
**Note**: Some base images have built-in users (node:node, bun:bun)

### 5. Secrets Exposure
```dockerfile
# 🔴 CRITICAL: Secrets baked in image
ENV AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
ARG DATABASE_PASSWORD=secret123
```
**Check**: ENV/ARG containing key, secret, password, token, credential → 🔴 Critical
**Fix**: Use runtime env vars or BuildKit secrets

### 6. .dockerignore
**Check**: No `.dockerignore` file exists → 🟡 Warning
**Minimum .dockerignore**:
```
.git
node_modules
npm-debug.log
.env
.env.*
coverage
*.md
.vscode
.idea
__pycache__
*.pyc
.pytest_cache
```

### 7. Health Checks
```dockerfile
# 🟢 GOOD: Meaningful health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```
**Check**: No `HEALTHCHECK` instruction → 🟡 Warning (unless Kubernetes)

### 8. Multi-Stage Builds
```dockerfile
# 🟢 GOOD: Separate build and runtime
FROM node:20-alpine AS builder
RUN npm ci && npm run build

FROM node:20-alpine AS production
COPY --from=builder /app/dist ./dist
```
**Check**: Single-stage with build tools in final image → 🟡 Warning

### 9. Layer Optimization
```dockerfile
# 🔴 BAD: Cleanup in separate layer (doesn't save space)
RUN apt-get update
RUN apt-get install -y curl wget
RUN apt-get clean

# 🟢 GOOD: Single layer with cleanup
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl wget && \
    rm -rf /var/lib/apt/lists/*
```
**Check**: Multiple consecutive `RUN apt-get` or cleanup in separate `RUN` → 🟡 Warning

### 10. Base Image Currency
**Check**: Base image older than 6 months → 🟡 Warning
**Action**: Run `docker scout cves <image>` or `trivy image <image>`

---

## Report Format

```markdown
# Docker Audit Report

## Summary
- 🔴 Critical: X issues
- 🟡 Warning: X issues
- 🟢 Good: X checks passed

## Findings

### 🔴 [Critical] Unpinned Base Image
**File**: Dockerfile:1
**Current**: `FROM node:latest`
**Fix**: `FROM node:20.11.1-alpine3.19`
**Impact**: Non-reproducible builds, surprise breakages

### 🟡 [Warning] Missing .dockerignore
**Impact**: Bloated build context, slower builds
**Fix**: Create .dockerignore with standard exclusions

## Recommended Actions (Priority Order)
1. Pin base image versions
2. Add .dockerignore
3. Reorder COPY for cache optimization
4. Add non-root user
```

---

## Quick Fixes

### Convert to Multi-Stage (Node.js)
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

### Convert to Multi-Stage (Python)
```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt
COPY . .

# Production stage
FROM python:3.12-slim AS production
WORKDIR /app
RUN useradd -r -s /bin/false appuser
COPY --from=builder --chown=appuser:appuser /app/deps /app/deps
COPY --from=builder --chown=appuser:appuser /app/src ./src
ENV PYTHONPATH=/app/deps
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
CMD ["python", "src/main.py"]
```

### Convert to Multi-Stage (Go)
```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# Production stage (distroless)
FROM gcr.io/distroless/static-debian12 AS production
COPY --from=builder /app/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Convert to Bun
```dockerfile
FROM oven/bun:1 AS builder
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM oven/bun:1-slim AS production
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER bun
EXPOSE 3000
CMD ["bun", "run", "dist/index.js"]
```

---

## Standard .dockerignore

```
# Git
.git
.gitignore

# Dependencies (reinstalled in container)
node_modules
vendor
.venv
__pycache__
*.pyc

# Build artifacts
dist
build
coverage
.nyc_output

# IDE/Editor
.vscode
.idea
*.swp
*.swo

# Environment/Secrets
.env
.env.*
*.pem
*.key

# Documentation
*.md
docs
LICENSE

# Tests (unless needed)
tests
__tests__
*.test.js
*.spec.js

# Docker
Dockerfile*
docker-compose*
.dockerignore

# CI/CD
.github
.gitlab-ci.yml
.circleci
```

---

## Workflow

1. **Find Dockerfiles**
   ```bash
   find . -name "Dockerfile*" -o -name "*.dockerfile"
   ```

2. **Run Audit**
   - Read each Dockerfile
   - Check against 10-point checklist
   - Generate findings report

3. **Measure Current State**
   ```bash
   docker build -t audit-before . && docker images audit-before
   docker history audit-before
   ```

4. **Apply Fixes** (with confirmation)

5. **Verify Improvement**
   ```bash
   docker build -t audit-after . && docker images audit-after
   time docker build --no-cache -t timing-test .
   ```

---

## Important Rules

- **ALWAYS** pin base image versions (major.minor.patch)
- **ALWAYS** use minimal base images (alpine/slim/distroless)
- **ALWAYS** copy dependency files before source code
- **ALWAYS** run as non-root in production
- **NEVER** put secrets in Dockerfile (ENV, ARG, or inline)
- **NEVER** use `COPY . .` before installing dependencies
- **NEVER** skip .dockerignore

---

## Size Targets

| Stack | Target Size | Base Image |
|-------|-------------|------------|
| Node.js | < 200MB | node:XX-alpine |
| Python | < 300MB | python:XX-slim |
| Go | < 20MB | distroless/static |
| Rust | < 20MB | distroless/cc |
| Java | < 300MB | eclipse-temurin:XX-jre-alpine |
