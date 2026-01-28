---
name: docker-optimize
description: "Audit and optimize Dockerfiles and docker-compose files for size, security, build speed, and best practices. Triggers on: optimize dockerfile, audit docker, fix dockerfile, docker best practices, docker compose security."
---

# Docker Optimization

Audit Dockerfiles and docker-compose files for common mistakes and security vulnerabilities. Based on patterns from hundreds of real-world Docker deployments and CWE-mapped security rules.

---

## The Job

1. Find Dockerfile(s) and docker-compose files in the project
2. Audit against security checklist (Critical → High → Medium)
3. Audit against optimization checklist
4. Generate prioritized findings report with CWE references
5. Apply fixes (with user confirmation)

---

## Part 1: Dockerfile Security Audit

Score: 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Good

### SEC-001: Docker Socket Exposure
**Severity**: 🔴 Critical | **CWE**: CWE-250
```dockerfile
# 🔴 CRITICAL: Full host compromise possible
VOLUME ["/var/run/docker.sock"]

# Also check for:
-v /var/run/docker.sock:/var/run/docker.sock
```
**Impact**: Grants full control over host Docker daemon. Complete container escape.
**Fix**: Never mount Docker socket. Use Docker API with authentication if needed.

### SEC-002: Running as Root
**Severity**: 🟠 High | **CWE**: CWE-250
```dockerfile
# 🟠 BAD: No USER instruction
CMD ["node", "server.js"]

# 🟢 GOOD: Explicit non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser
CMD ["node", "server.js"]
```
**Check**: No `USER` instruction before final `CMD`/`ENTRYPOINT`

### SEC-003: Secrets in ENV/ARG
**Severity**: 🔴 Critical | **CWE**: CWE-538
```dockerfile
# 🔴 CRITICAL: Visible in docker history forever
ENV AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
ARG DATABASE_PASSWORD=secret123

# 🟢 GOOD: BuildKit secrets (never persisted)
RUN --mount=type=secret,id=db_password,target=/run/secrets/db_password \
    cat /run/secrets/db_password | some-command
```
**Check**: ENV/ARG containing: key, secret, password, token, credential, api_key
**Note**: ARG values visible in `docker history` even without default values

### SEC-004: Sudo in Dockerfile
**Severity**: 🟡 Medium | **CWE**: CWE-250
```dockerfile
# 🟡 BAD: Unnecessary privilege escalation
RUN sudo apt-get install -y curl

# 🟢 GOOD: Run as root before USER, then drop privileges
RUN apt-get install -y curl
USER appuser
```
**Fix**: Run privileged commands before `USER` instruction, not with sudo

### SEC-005: ADD vs COPY
**Severity**: 🟡 Medium
```dockerfile
# 🟡 BAD: ADD has hidden behaviors (auto-extract, remote fetch)
ADD https://example.com/file.tar.gz /app/
ADD archive.tar.gz /app/

# 🟢 GOOD: COPY is explicit and predictable
COPY archive.tar.gz /app/
RUN tar -xzf /app/archive.tar.gz
```
**Check**: Any `ADD` instruction → prefer `COPY` unless extraction needed

### SEC-006: Shell Form vs Exec Form
**Severity**: 🟡 Medium
```dockerfile
# 🟡 BAD: Shell form - no signal handling, extra shell process
CMD npm start
ENTRYPOINT /app/start.sh

# 🟢 GOOD: Exec form - proper PID 1, signal handling
CMD ["npm", "start"]
ENTRYPOINT ["/app/start.sh"]
```
**Impact**: Shell form prevents proper SIGTERM handling, graceful shutdown fails

### SEC-007: Missing pipefail
**Severity**: 🟡 Medium
```dockerfile
# 🟡 BAD: Pipe failure ignored
RUN curl -s https://example.com/script.sh | bash

# 🟢 GOOD: Fail on any pipe component failure
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN curl -s https://example.com/script.sh | bash
```
**Check**: RUN with pipes (`|`) without `set -o pipefail` or SHELL override

---

## Part 2: Docker Compose Security Audit

### COMPOSE-SEC-001: Privileged Mode
**Severity**: 🔴 Critical | **CWE**: CWE-250
```yaml
# 🔴 CRITICAL: Disables ALL container isolation
services:
  app:
    privileged: true
```
**Impact**: Access to all devices, can load kernel modules, complete host escape
**Fix**: Remove `privileged: true`. Use specific capabilities if needed.

### COMPOSE-SEC-002: Docker Socket Mount
**Severity**: 🔴 Critical | **CWE**: CWE-250
```yaml
# 🔴 CRITICAL: Full Docker daemon access
services:
  app:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
**Fix**: Remove socket mount. Use Docker API with TLS authentication.

### COMPOSE-SEC-003: Seccomp Disabled
**Severity**: 🟠 High | **CWE**: CWE-284
```yaml
# 🟠 BAD: Removes syscall filtering
services:
  app:
    security_opt:
      - seccomp:unconfined
```
**Fix**: Remove or use custom seccomp profile: `seccomp:./custom-seccomp.json`

### COMPOSE-SEC-004: Host Network Mode
**Severity**: 🟠 High | **CWE**: CWE-250
```yaml
# 🟠 BAD: Bypasses network isolation
services:
  app:
    network_mode: host
```
**Impact**: Container shares host network stack, can access all host ports
**Fix**: Use bridge network with explicit port mappings

### COMPOSE-SEC-005: Dangerous Capabilities
**Severity**: 🟠 High | **CWE**: CWE-250
```yaml
# 🟠 BAD: Overly permissive capabilities
services:
  app:
    cap_add:
      - SYS_ADMIN    # Near-root powers
      - NET_ADMIN    # Network manipulation
      - SYS_PTRACE   # Process debugging
      - ALL          # Everything

# 🟢 GOOD: Drop all, add only what's needed
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if binding <1024
```

### COMPOSE-SEC-006: Host PID/IPC Namespace
**Severity**: 🟠 High | **CWE**: CWE-250
```yaml
# 🟠 BAD: Shares host process/IPC namespace
services:
  app:
    pid: host
    ipc: host
```
**Impact**: Can see/signal all host processes, access shared memory
**Fix**: Remove `pid: host` and `ipc: host`

### COMPOSE-SEC-007: Missing no-new-privileges
**Severity**: 🟡 Medium | **CWE**: CWE-732
```yaml
# 🟢 GOOD: Prevent privilege escalation
services:
  app:
    security_opt:
      - no-new-privileges:true
```
**Check**: Missing `no-new-privileges:true` in security_opt

### COMPOSE-SEC-008: SELinux/AppArmor Disabled
**Severity**: 🟡 Medium | **CWE**: CWE-732
```yaml
# 🟡 BAD: Disables mandatory access controls
services:
  app:
    security_opt:
      - label:disable
      - apparmor:unconfined
```

### COMPOSE-SEC-009: Writable Root Filesystem
**Severity**: 🟡 Medium
```yaml
# 🟢 GOOD: Read-only root, explicit writable paths
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    volumes:
      - app-data:/app/data
```
**Check**: Missing `read_only: true`

---

## Part 3: Optimization Checklist

### OPT-001: Base Image Tags
```dockerfile
# 🔴 BAD: Moving target
FROM node:latest

# 🟢 GOOD: Pinned version
FROM node:20.11.1-alpine3.19
```

### OPT-002: Image Size
```dockerfile
# 🟡 BAD: Full OS (1.8GB+)
FROM ubuntu:22.04

# 🟢 GOOD: Minimal base
FROM node:20-alpine      # ~180MB
FROM python:3.12-slim    # ~150MB
FROM gcr.io/distroless/static  # ~2MB
```

### OPT-003: Layer Cache
```dockerfile
# 🔴 BAD: Cache busted on every change
COPY . .
RUN npm install

# 🟢 GOOD: Dependencies cached
COPY package*.json ./
RUN npm ci
COPY . .
```

### OPT-004: Package Manager Flags
```dockerfile
# 🟡 BAD: Missing optimization flags
RUN apt-get update && apt-get install -y curl
RUN apk add curl
RUN pip install requests

# 🟢 GOOD: With cache/size optimization
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
RUN apk add --no-cache curl
RUN pip install --no-cache-dir requests
```

### OPT-005: apt-get upgrade
```dockerfile
# 🟡 BAD: Non-reproducible, adds bloat
RUN apt-get update && apt-get upgrade -y

# 🟢 GOOD: Pin base image version instead
FROM debian:12.4-slim
```
**Why**: Upgrades are unpredictable. Pin your base image for reproducibility.

### OPT-006: Multi-Stage Builds
```dockerfile
# 🟢 GOOD: Build tools don't ship to production
FROM node:20-alpine AS builder
RUN npm ci && npm run build

FROM node:20-alpine AS production
COPY --from=builder /app/dist ./dist
```

### OPT-007: .dockerignore
**Check**: No `.dockerignore` file
**Impact**: Bloated context, slower builds, potential secret leakage

### OPT-008: Health Checks
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

---

## Report Format

```markdown
# Docker Security & Optimization Audit

## Summary
- 🔴 Critical: X issues (container escape risk)
- 🟠 High: X issues (privilege escalation risk)
- 🟡 Medium: X issues (best practice violations)
- 🟢 Good: X checks passed

## Security Findings

### 🔴 [CRITICAL] Docker Socket Exposed (SEC-001)
**File**: docker-compose.yml:12
**CWE**: CWE-250 (Execution with Unnecessary Privileges)
**Current**: `- /var/run/docker.sock:/var/run/docker.sock`
**Impact**: Complete host compromise possible
**Fix**: Remove socket mount, use Docker API with TLS

### 🟠 [HIGH] Privileged Container (COMPOSE-SEC-001)
**File**: docker-compose.yml:8
**CWE**: CWE-250
**Fix**: Remove `privileged: true`, use specific capabilities

## Optimization Findings

### 🟡 [MEDIUM] Unpinned Base Image (OPT-001)
**File**: Dockerfile:1
**Current**: `FROM node:latest`
**Fix**: `FROM node:20.11.1-alpine3.19`

## Recommended Actions (Priority Order)
1. 🔴 Remove Docker socket mount
2. 🔴 Remove hardcoded secrets
3. 🟠 Disable privileged mode
4. 🟠 Add non-root user
5. 🟡 Pin base image versions
```

---

## Secure docker-compose.yml Template

```yaml
version: "3.8"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:${VERSION:-latest}

    # Security hardening
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if needed

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          memory: 256M

    # Networking
    networks:
      - app-network
    ports:
      - "3000:3000"

    # Writable paths only where needed
    tmpfs:
      - /tmp:size=100M
    volumes:
      - app-data:/app/data:rw

    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s

    # Environment (no secrets here)
    environment:
      - NODE_ENV=production
    env_file:
      - .env  # Secrets in .env, not committed

networks:
  app-network:
    driver: bridge

volumes:
  app-data:
```

---

## Quick Reference: Dockerfile Template

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20.11.1-alpine AS builder

WORKDIR /app

# Dependencies first (cache optimization)
COPY package*.json ./
RUN npm ci

# Source code
COPY . .
RUN npm run build

# Production stage
FROM node:20.11.1-alpine AS production

WORKDIR /app
ENV NODE_ENV=production

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Production dependencies only
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Built artifacts
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

---

## Workflow

1. **Find files**
   ```bash
   find . -name "Dockerfile*" -o -name "docker-compose*.yml" -o -name "compose*.yml"
   ```

2. **Run security audit first** (Critical → High → Medium)

3. **Run optimization audit**

4. **Scan for CVEs**
   ```bash
   docker scout cves myapp:latest
   trivy image myapp:latest
   ```

5. **Apply fixes** (with user confirmation)

6. **Verify**
   ```bash
   docker build -t audit-after .
   docker images audit-after
   docker history audit-after
   ```

---

## Important Rules

### Security (Never Violate)
- **NEVER** mount Docker socket
- **NEVER** use `privileged: true`
- **NEVER** put secrets in Dockerfile/docker-compose
- **NEVER** run as root in production
- **NEVER** disable seccomp/AppArmor without justification

### Optimization (Best Practice)
- **ALWAYS** pin base image versions
- **ALWAYS** use minimal base images
- **ALWAYS** copy dependency files before source
- **ALWAYS** use exec form for CMD/ENTRYPOINT
- **ALWAYS** include .dockerignore
- **ALWAYS** use `--no-install-recommends` / `--no-cache` / `--no-cache-dir`

---

## Size Targets

| Stack | Target | Base Image |
|-------|--------|------------|
| Node.js | < 200MB | node:XX-alpine |
| Python | < 300MB | python:XX-slim |
| Go | < 20MB | distroless/static |
| Rust | < 20MB | distroless/cc |
| Java | < 300MB | eclipse-temurin:XX-jre-alpine |
| Bun | < 200MB | oven/bun:XX-slim |
