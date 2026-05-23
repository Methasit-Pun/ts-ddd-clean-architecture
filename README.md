# TypeScript DDD Clean Architecture — Claude Skills

A collection of Claude AI skills for building production-grade **TypeScript** applications using **Domain-Driven Design (DDD)**, **Hexagonal Architecture**, **Express**, **Prisma**, **Socket.IO**, and **Redis**.

Drop these skills into your `~/.claude/skills/` folder and Claude will automatically apply the right patterns when you ask for help with any of the covered topics.

---

## Skills Included

### Agent Roles

Skills are organized into three agent roles. The [`agents.md`](./skills/agents.md) manifest describes which skills each role always loads vs. loads on demand.

#### System Architect
| Skill | Description |
|---|---|
| [`system-architect/blueprint`](./skills/system-architect/blueprint/SKILL.md) | Turn a one-line objective into a layered, step-by-step construction plan across Domain → Application → Infrastructure → Presentation |
| [`system-architect/adr-writer`](./skills/system-architect/adr-writer/SKILL.md) | Write structured Architecture Decision Records (ADRs) capturing context, trade-offs, and outcomes |
| [`system-architect/bounded-context`](./skills/system-architect/bounded-context/SKILL.md) | Map bounded contexts, define ubiquitous language, and identify aggregates and domain events |

#### Backend Engineer
| Skill | Description |
|---|---|
| [`backend-engineer/cqrs`](./skills/backend-engineer/cqrs/SKILL.md) | Implement the CQRS pattern — Commands (write) and Queries (read) stay strictly separated in the Application layer |
| [`backend-engineer/repository-pattern`](./skills/backend-engineer/repository-pattern/SKILL.md) | Define repository interfaces in the Domain layer and wire Prisma/TypeORM/Drizzle implementations in Infrastructure |

#### DevOps / Platform Engineer
| Skill | Description |
|---|---|
| [`devops/ci-design`](./skills/devops/ci-design/SKILL.md) | Design GitHub Actions / GitLab CI pipelines, Docker builds, environment promotion, and secrets management |

#### Shared (load on demand)
| Skill | Description |
|---|---|
| [`_shared/code-review`](./skills/_shared/code-review/SKILL.md) | DDD-aware code review — correctness, layer violations, and simplification |
| [`_shared/security-review`](./skills/_shared/security-review/SKILL.md) | OWASP-mapped security audit covering injection, auth/authz, data exposure, and misconfiguration |
| [`_shared/simplify`](./skills/_shared/simplify/SKILL.md) | Remove unnecessary complexity, eliminate duplication, and reduce premature abstractions |

---

### Standalone Skills

| Skill | Description |
|---|---|
| [`ts-ddd-clean-architecture`](./skills/ts-ddd-clean-architecture/SKILL.md) | Hexagonal Architecture, DDD Entities, Value Objects, Domain Events, Use Cases, Ports & Adapters |
| [`docker-redis-iac`](./skills/docker-redis-iac/SKILL.md) | Multi-stage Dockerfiles, Docker Compose, Socket.IO Redis adapter, safe Prisma migrations |
| [`tdd-domain-layer`](./skills/tdd-domain-layer/SKILL.md) | TDD with Vitest — Red-Green-Refactor for Domain and Application layers, mocking ports |
| [`zod-express-validation`](./skills/zod-express-validation/SKILL.md) | Runtime validation with Zod, `z.infer` DTOs, Express middleware, Socket.IO payload guards |
| [`agent-orchestration`](./skills/agent-orchestration/SKILL.md) | AST-based code generation with `ts-morph`, Prisma schema parsing, Handlebars scaffolding scripts |
| [`cicd-fix-deploy`](./skills/cicd-fix-deploy/SKILL.md) | Fix failing GitHub Actions workflows, TypeScript errors, and Vitest failures; pre-deploy readiness gate |

---

## Installation

### Option A — Clone and copy (recommended)

```bash
# Clone the repo
git clone https://github.com/Methasit-Pun/ts-ddd-clean-architecture.git

# Copy all skills into your Claude skills directory
cp -r ts-ddd-clean-architecture/skills/* ~/.claude/skills/
```

### Option B — Copy a single skill

```bash
# Example: copy only the CQRS skill
cp -r ts-ddd-clean-architecture/skills/backend-engineer/cqrs ~/.claude/skills/backend-engineer/cqrs
```

### Option C — Download a single SKILL.md manually

Click any skill link in the table above → click **Raw** → save as `~/.claude/skills/<skill-path>/SKILL.md`.

---

## How Skills Work

Each skill is a `SKILL.md` file stored at `~/.claude/skills/<skill-name>/SKILL.md`.

Claude reads the skill when you describe a task that matches its **"When to Activate"** section — no extra configuration needed. The skill provides:

- Concrete code examples for every pattern
- A step-by-step execution workflow
- An anti-patterns table to prevent common mistakes

The [`agents.md`](./skills/agents.md) manifest lets you compose multi-skill agents by declaring which skills a role always loads vs. loads on demand.

---

## Repo Structure

```
skills/
├── agents.md                        ← Agent role manifest
├── _shared/
│   ├── code-review/SKILL.md         ← DDD-aware code review
│   ├── security-review/SKILL.md     ← OWASP security audit
│   └── simplify/SKILL.md            ← Complexity reduction
├── system-architect/
│   ├── blueprint/SKILL.md           ← Construction planning
│   ├── adr-writer/SKILL.md          ← Architecture Decision Records
│   └── bounded-context/SKILL.md    ← Domain mapping
├── backend-engineer/
│   ├── cqrs/SKILL.md                ← Command/Query separation
│   └── repository-pattern/SKILL.md ← Persistence abstraction
├── devops/
│   └── ci-design/SKILL.md           ← CI/CD pipeline design
├── ts-ddd-clean-architecture/
│   └── SKILL.md                     ← Hexagonal Architecture + DDD
├── docker-redis-iac/
│   └── SKILL.md                     ← Docker, Redis, production deployments
├── tdd-domain-layer/
│   └── SKILL.md                     ← TDD with Vitest, mocking ports
├── zod-express-validation/
│   └── SKILL.md                     ← Zod schemas, Express middleware
├── agent-orchestration/
│   └── SKILL.md                     ← Code generation scripts, ts-morph
└── cicd-fix-deploy/
    └── SKILL.md                     ← Fix CI failures, pre-deploy gate
```

---

## Stack Coverage

- **Language:** TypeScript
- **Runtime:** Node.js 20+
- **API:** Express
- **Real-time:** Socket.IO
- **ORM:** Prisma (PostgreSQL)
- **Caching / PubSub:** Redis (`ioredis`, `@socket.io/redis-adapter`)
- **Testing:** Vitest
- **Validation:** Zod
- **Containerisation:** Docker, Docker Compose
- **CI/CD:** GitHub Actions, GitLab CI

---

## Contributing

1. Fork the repo
2. Add or improve a skill under the appropriate folder (`system-architect/`, `backend-engineer/`, `devops/`, `_shared/`, or a new standalone folder)
3. Follow the existing SKILL.md format (frontmatter → When to Activate → code examples → anti-patterns)
4. Update `agents.md` if the skill belongs to an agent role
5. Open a PR

---

## License

MIT
