# Docker Optimize Skill

A [Claude Code](https://claude.ai/claude-code) skill that audits Dockerfiles and docker-compose files for security vulnerabilities, size optimization, and best practices.

## Installation

```bash
git clone https://github.com/vnnkl/docker-optimize-skill ~/.claude/skills/docker-optimize
```

## Usage

In Claude Code, trigger the skill with:
- "optimize my dockerfile"
- "audit docker"
- "docker compose security"
- "fix dockerfile"

Or invoke directly: `/docker-optimize`

## What It Checks

### Security Audit (19 rules with CWE mappings)

**Dockerfile Security (10 rules):**
| Rule | Severity | Issue |
|------|----------|-------|
| SEC-001 | 🔴 Critical | Docker socket exposure |
| SEC-002 | 🟠 High | Running as root (58% of images!) |
| SEC-003 | 🔴 Critical | Secrets in ENV/ARG |
| SEC-004 | 🟡 Medium | Sudo usage |
| SEC-005 | 🟡 Medium | ADD instead of COPY |
| SEC-006 | 🟡 Medium | Shell form CMD/ENTRYPOINT |
| SEC-007 | 🟡 Medium | Missing pipefail |
| SEC-008 | 🟡 Medium | Hardcoded UID (breaks OpenShift) |
| SEC-009 | 🟡 Medium | Writable binaries |
| SEC-010 | 🔴 Critical | Secrets "deleted" but in layer |

**Docker Compose Security (9 rules):**
| Rule | Severity | Issue |
|------|----------|-------|
| COMPOSE-SEC-001 | 🔴 Critical | Privileged mode |
| COMPOSE-SEC-002 | 🔴 Critical | Docker socket mount |
| COMPOSE-SEC-003 | 🟠 High | Seccomp disabled |
| COMPOSE-SEC-004 | 🟠 High | Host network mode |
| COMPOSE-SEC-005 | 🟠 High | Dangerous capabilities |
| COMPOSE-SEC-006 | 🟠 High | Host PID/IPC namespace |
| COMPOSE-SEC-007 | 🟡 Medium | Missing no-new-privileges |
| COMPOSE-SEC-008 | 🟡 Medium | SELinux/AppArmor disabled |
| COMPOSE-SEC-009 | 🟡 Medium | Writable root filesystem |

### Optimization Audit (13 rules)

| Rule | Issue |
|------|-------|
| OPT-001 | Unpinned base image tags (use digests for prod) |
| OPT-002 | Bloated base images (prefer distroless) |
| OPT-003 | Cache-busting layer order |
| OPT-004 | Missing package manager flags |
| OPT-005 | Using apt-get upgrade |
| OPT-006 | Missing multi-stage builds |
| OPT-007 | No .dockerignore |
| OPT-008 | Missing health checks |
| OPT-009 | ENV/EXPOSE placement (cache optimization) |
| OPT-010 | Multiple FROM pitfall |
| OPT-011 | VOLUME timing issues |
| OPT-012 | Layer consolidation |
| OPT-013 | Missing OCI labels |

## Workflow

1. Find Dockerfiles and docker-compose files
2. Run Hadolint for static analysis
3. Security audit (Critical → High → Medium)
4. Optimization audit
5. CVE scanning (docker scout / trivy)
6. Apply fixes with confirmation

## Output

Generates a prioritized findings report with:
- Summary by severity (Critical → High → Medium)
- CWE references for compliance
- File locations and line numbers
- Ready-to-use fix code
- Secure templates for Dockerfile and docker-compose.yml

## Size Targets

| Stack | Target | Recommended Base |
|-------|--------|------------------|
| Node.js | < 200MB | `node:XX-alpine` |
| Python | < 300MB | `python:XX-slim` |
| Go | < 20MB | `distroless/static` |
| Bun | < 200MB | `oven/bun:XX-slim` |

## Credits

- [Sysdig: Top 20 Dockerfile Best Practices](https://www.sysdig.com/learn-cloud-native/dockerfile-best-practices)
- [Atlassian: Common Dockerfile Mistakes](https://www.atlassian.com/blog/developer/common-dockerfile-mistakes)
- [CodePathfinder: Docker Compose Security Rules](https://codepathfinder.dev/blog/announcing-docker-compose-security-rules)
- [Hadolint: Dockerfile Linter](https://github.com/hadolint/hadolint)

## License

MIT
