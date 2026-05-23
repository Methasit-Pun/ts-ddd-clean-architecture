# Agent Skill Manifest

Compose an agent by combining its role-specific skills. Shared skills are **load on demand** — only read them when the task explicitly requires it, not on every invocation.

---

## System Architect

**Responsibility:** Domain design, bounded context mapping, architectural decisions, project planning.

**Always load:**
| Skill | Path |
|-------|------|
| Blueprint | `system-architect/blueprint` |
| ADR Writer | `system-architect/adr-writer` |
| Bounded Context | `system-architect/bounded-context` |

**Load on demand:**
| Skill | Path | When |
|-------|------|------|
| Code Review | `_shared/code-review` | User asks for review or layer-violation check |
| Simplify | `_shared/simplify` | User says "too complex" or asks for over-engineering check |
| Security Review | `_shared/security-review` | Pre-deploy audit or explicit security ask |

---

## Backend Engineer

**Responsibility:** Application and domain layer implementation, persistence wiring, use cases.

**Always load:**
| Skill | Path |
|-------|------|
| CQRS | `backend-engineer/cqrs` |
| Repository Pattern | `backend-engineer/repository-pattern` |

**Load on demand:**
| Skill | Path | When |
|-------|------|------|
| Code Review | `_shared/code-review` | Review requested or PR ready for check |
| Simplify | `_shared/simplify` | User says "too much boilerplate" or asks to reduce code |
| Security Review | `_shared/security-review` | Auth, input validation, or data exposure is in scope |

---

## DevOps / Platform Engineer

**Responsibility:** CI/CD pipelines, Docker, environment promotion, secrets, observability.

**Always load:**
| Skill | Path |
|-------|------|
| CI Design | `devops/ci-design` |

**Load on demand:**
| Skill | Path | When |
|-------|------|------|
| Security Review | `_shared/security-review` | Pipeline secrets, Docker config, or exposed endpoint audit |
| Code Review | `_shared/code-review` | Infrastructure-as-code review requested |

---

## Shared Skills

Load these only when the task explicitly calls for them. Never load all three by default.

| Skill | Path | Purpose |
|-------|------|---------|
| Code Review | `_shared/code-review` | DDD-aware correctness and layer discipline |
| Security Review | `_shared/security-review` | OWASP-mapped security audit |
| Simplify | `_shared/simplify` | Remove unnecessary complexity and premature abstractions |

---

## Dependency rule (enforced by all agents)

```
Domain  ←  Application  ←  Infrastructure  ←  Presentation
```

Lower layers never import from higher. No exceptions without a documented ADR.

---

## Skill folder structure

```
skills/
  agents.md
  _shared/
    code-review/SKILL.md
    security-review/SKILL.md
    simplify/SKILL.md
  system-architect/
    blueprint/SKILL.md
    adr-writer/SKILL.md
    bounded-context/SKILL.md
  backend-engineer/
    cqrs/SKILL.md
    repository-pattern/SKILL.md
  devops/
    ci-design/SKILL.md
```
