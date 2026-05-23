# ts-ddd-clean-architecture

This repo is a **Claude skills distribution library** — a collection of SKILL.md files that users install into `~/.claude/skills/`. There is no application source code here. All work in this repo is authoring, editing, or reviewing skills.

## What this repo contains

```
skills/
  _shared/          ← Skills used by multiple agent roles (load on demand)
  system-architect/ ← Skills for domain design and architectural decisions
  backend-engineer/ ← Skills for application/domain/infrastructure implementation
  devops/           ← Skills for CI/CD, Docker, environment management
  agents.md         ← Manifest defining which skills each agent role loads
```

## Stack the skills cover

- **Language:** TypeScript (Node.js 20+)
- **API:** Express
- **Real-time:** Socket.IO
- **ORM:** Prisma (PostgreSQL)
- **Cache / PubSub:** Redis (`ioredis`, `@socket.io/redis-adapter`)
- **Testing:** Vitest
- **Validation:** Zod
- **Container:** Docker, Docker Compose

## DDD layer dependency rule (enforced by all skills)

```
Domain  ←  Application  ←  Infrastructure  ←  Presentation
```

- **Domain** imports nothing outside `domain/`
- **Application** imports domain only
- **Infrastructure** imports domain + application interfaces
- **Presentation** imports application only (never domain directly)

Any import that violates this direction is a blocker. No exceptions without a documented ADR.

## SKILL.md authoring conventions

Every skill file must have:
```yaml
---
name: kebab-case-name
description: One sentence that tells the harness WHEN to trigger this skill. Be specific — vague descriptions cause the skill to fire on wrong tasks or never fire.
---
```

Body structure:
- Lead with activation conditions and core principles
- Use checklists for review tasks, workflow steps for implementation tasks
- No code examples unless the pattern is non-obvious or frequently done wrong
- No "What NOT to do" sections for things linters or TypeScript already catch
- Target under 70 lines per SKILL.md — long skills bloat the context window every time they are loaded

## Shared skills loading policy

Skills in `_shared/` are **on-demand only** — they are not loaded by default for any agent. An agent reads a shared skill only when the task explicitly requires it (e.g., user asks for a code review, security audit, or simplification pass). See `skills/agents.md` for the full manifest.
