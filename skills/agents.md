# Agent Skill Manifest

This file defines which skills each agent role loads. Compose an agent by combining the shared skills with its role-specific skills.

---

## System Architect

**Responsibility:** Domain design, bounded context mapping, architectural decisions, project planning.

**Skills:**
| Skill | Path | When to use |
|-------|------|-------------|
| Blueprint | `system-architect/blueprint` | Plan a new feature or system across DDD layers |
| ADR Writer | `system-architect/adr-writer` | Document a technology or design decision |
| Bounded Context | `system-architect/bounded-context` | Map domain boundaries, aggregates, ubiquitous language |
| Code Review | `_shared/code-review` | Review any code for layer violations and DDD correctness |
| Security Review | `_shared/security-review` | Security audit before any significant design is shipped |
| Simplify | `_shared/simplify` | Identify over-engineering in proposed architecture |

---

## Backend Engineer

**Responsibility:** Application and domain layer implementation, persistence wiring, use cases.

**Skills:**
| Skill | Path | When to use |
|-------|------|-------------|
| CQRS | `backend-engineer/cqrs` | Implement commands, queries, handlers, and event flow |
| Repository Pattern | `backend-engineer/repository-pattern` | Implement persistence with ORM mapper and domain interface |
| Code Review | `_shared/code-review` | Review domain/application/infrastructure code |
| Security Review | `_shared/security-review` | Security check on auth, input validation, data exposure |
| Simplify | `_shared/simplify` | Reduce boilerplate and premature abstractions |

---

## DevOps / Platform Engineer

**Responsibility:** CI/CD pipelines, Docker, environment promotion, secrets, observability.

**Skills:**
| Skill | Path | When to use |
|-------|------|-------------|
| CI Design | `devops/ci-design` | Design or update a CI/CD pipeline |
| Security Review | `_shared/security-review` | Audit pipeline secrets, Docker config, exposed endpoints |
| Code Review | `_shared/code-review` | Review infrastructure-as-code changes |

---

## Shared Skills (available to all agents)

| Skill | Path | Purpose |
|-------|------|---------|
| Code Review | `_shared/code-review` | DDD-aware correctness and layer discipline review |
| Security Review | `_shared/security-review` | OWASP-mapped security audit |
| Simplify | `_shared/simplify` | Remove unnecessary complexity and premature abstractions |

---

## Dependency rule (enforced by all agents)

```
Domain  ←  Application  ←  Infrastructure  ←  Presentation
```

Every agent must respect this direction. Lower layers never import from higher layers.
No exceptions without a documented ADR.

---

## Skill folder structure

```
.claude/skills/
  agents.md                          ← this file
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
