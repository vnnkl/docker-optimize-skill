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

### Security Audit (17 rules with CWE mappings)

**Dockerfile Security:**
| Rule | Severity | Issue |
|------|----------|-------|
| SEC-001 | 🔴 Critical | Docker socket exposure |
| SEC-002 | 🟠 High | Running as root |
| SEC-003 | 🔴 Critical | Secrets in ENV/ARG |
| SEC-004 | 🟡 Medium | Sudo usage |
| SEC-005 | 🟡 Medium | ADD instead of COPY |
| SEC-006 | 🟡 Medium | Shell form CMD/ENTRYPOINT |
| SEC-007 | 🟡 Medium | Missing pipefail |

**Docker Compose Security:**
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

### Optimization Audit (8 rules)

| Rule | Issue |
|------|-------|
| OPT-001 | Unpinned base image tags (`:latest`) |
| OPT-002 | Bloated base images |
| OPT-003 | Cache-busting layer order |
| OPT-004 | Missing package manager flags |
| OPT-005 | Using apt-get upgrade |
| OPT-006 | Missing multi-stage builds |
| OPT-007 | No .dockerignore |
| OPT-008 | Missing health checks |

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

Security rules based on [CodePathfinder's Docker security rules](https://codepathfinder.dev/blog/announcing-docker-compose-security-rules).

## License

MIT
