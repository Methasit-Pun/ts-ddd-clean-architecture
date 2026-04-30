# TypeScript DDD Clean Architecture — Claude Skills

A collection of Claude AI skills for building production-grade **TypeScript** applications using **Domain-Driven Design (DDD)**, **Hexagonal Architecture**, **Express**, **Prisma**, **Socket.IO**, and **Redis**.

Drop these skills into your `~/.claude/skills/` folder and Claude will automatically apply the right patterns when you ask for help with any of the covered topics.

---

## Skills Included

| Skill | Description |
|---|---|
| [`ts-ddd-clean-architecture`](./skills/ts-ddd-clean-architecture/SKILL.md) | Hexagonal Architecture, DDD Entities, Value Objects, Domain Events, Use Cases, Ports & Adapters |
| [`docker-redis-iac`](./skills/docker-redis-iac/SKILL.md) | Multi-stage Dockerfiles, Docker Compose, Socket.IO Redis adapter, safe Prisma migrations |
| [`tdd-domain-layer`](./skills/tdd-domain-layer/SKILL.md) | TDD with Vitest — Red-Green-Refactor for Domain and Application layers, mocking ports |
| [`zod-express-validation`](./skills/zod-express-validation/SKILL.md) | Runtime validation with Zod, `z.infer` DTOs, Express middleware, Socket.IO payload guards |
| [`agent-orchestration`](./skills/agent-orchestration/SKILL.md) | AST-based code generation with `ts-morph`, Prisma schema parsing, Handlebars scaffolding scripts |

---

## Installation

### Option A — Clone and copy (recommended)

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/ts-ddd-clean-architecture.git

# Copy all skills into your Claude skills directory
cp -r ts-ddd-clean-architecture/skills/* ~/.claude/skills/
```

### Option B — Copy a single skill

```bash
# Example: copy only the DDD architecture skill
cp -r ts-ddd-clean-architecture/skills/ts-ddd-clean-architecture ~/.claude/skills/
```

### Option C — Download a single SKILL.md manually

Click any skill link in the table above → click **Raw** → save as `~/.claude/skills/<skill-name>/SKILL.md`.

---

## How Skills Work

Each skill is a `SKILL.md` file stored at `~/.claude/skills/<skill-name>/SKILL.md`.

Claude reads the skill when you describe a task that matches its **"When to Activate"** section — no extra configuration needed. The skill provides:

- Concrete code examples for every pattern
- A step-by-step execution workflow
- An anti-patterns table to prevent common mistakes

---

## Repo Structure

```
skills/
├── ts-ddd-clean-architecture/
│   └── SKILL.md        ← Hexagonal Architecture + DDD
├── docker-redis-iac/
│   └── SKILL.md        ← Docker, Redis, production deployments
├── tdd-domain-layer/
│   └── SKILL.md        ← TDD with Vitest, mocking ports
├── zod-express-validation/
│   └── SKILL.md        ← Zod schemas, Express middleware, Socket.IO validation
└── agent-orchestration/
    └── SKILL.md        ← Code generation scripts, ts-morph, Handlebars
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

---

## Contributing

1. Fork the repo
2. Add or improve a skill under `skills/<skill-name>/SKILL.md`
3. Follow the existing SKILL.md format (frontmatter → When to Activate → code examples → anti-patterns)
4. Open a PR

---

## License

MIT
