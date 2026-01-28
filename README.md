# Docker Optimize Skill

A [Claude Code](https://claude.ai/claude-code) skill that audits and optimizes Dockerfiles for size, security, build speed, and best practices.

## Installation

```bash
# Clone to your Claude skills directory
git clone https://github.com/YOUR_USERNAME/docker-optimize-skill ~/.claude/skills/docker-optimize
```

Or add as a symlink:

```bash
git clone https://github.com/YOUR_USERNAME/docker-optimize-skill ~/Code/docker-optimize-skill
ln -s ~/Code/docker-optimize-skill ~/.claude/skills/docker-optimize
```

## Usage

In Claude Code, trigger the skill with:
- "optimize my dockerfile"
- "audit docker"
- "fix dockerfile"
- "docker best practices"

Or invoke directly: `/docker-optimize`

## What It Does

Audits Dockerfiles against 10 critical mistakes:

| Check | Severity | Issue |
|-------|----------|-------|
| Base Image Tags | 🔴 Critical | Using `:latest` or unpinned versions |
| Image Size | 🟡 Warning | Full OS images instead of alpine/slim |
| Layer Cache | 🔴 Critical | `COPY . .` before dependency install |
| Non-Root User | 🟡 Warning | Running as root |
| Secrets | 🔴 Critical | ENV/ARG with keys, passwords, tokens |
| .dockerignore | 🟡 Warning | Missing or incomplete |
| Health Checks | 🟡 Warning | No HEALTHCHECK instruction |
| Multi-Stage | 🟡 Warning | Build tools in production image |
| Layer Optimization | 🟡 Warning | Cleanup in separate RUN layers |
| Base Currency | 🟡 Warning | Outdated base images with CVEs |

## Output

Generates a prioritized findings report with:
- Summary of issues by severity
- Specific file locations and line numbers
- Ready-to-use fix code
- Before/after size comparison

## Size Targets

| Stack | Target | Recommended Base |
|-------|--------|------------------|
| Node.js | < 200MB | `node:XX-alpine` |
| Python | < 300MB | `python:XX-slim` |
| Go | < 20MB | `distroless/static` |
| Bun | < 200MB | `oven/bun:XX-slim` |

## License

MIT
