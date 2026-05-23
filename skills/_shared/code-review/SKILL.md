---
name: ts-ddd-code-review
description: Review TypeScript DDD clean architecture code for correctness bugs, layer violations, and simplification opportunities. Trigger when the user says "review this", "check this code", "code review", "what's wrong with this", "is this correct DDD?", "does this follow clean architecture?", or when changes touch domain, application, or infrastructure layers.
---

# Code Review — TypeScript DDD

## Layer dependency rule (check first — any violation is a [BLOCKER])

```
Domain    → imports nothing outside domain/
Application → imports domain only
Infrastructure → imports domain + application interfaces
Presentation → imports application only
```

## Checklist by layer

**Domain** (`src/*/domain/`)
- No framework imports (NestJS, TypeORM, Prisma, Express)
- Aggregates enforce invariants — business rules in aggregate, not handlers
- Value Objects are immutable (all properties `readonly`, no setters)
- Meaningful state changes emit domain events
- Repository interfaces use domain types only (no ORM entities, no raw strings)
- Flag anemic model if aggregate has only getters and no behavior methods

**Application** (`src/*/application/`)
- Handlers orchestrate only — no `if (order.status === 'PENDING')` domain logic
- Commands return void or ID, never a domain aggregate
- Queries use read models, not domain aggregates
- No infrastructure imports — depend on interfaces only

**Infrastructure** (`src/*/infrastructure/`)
- Mapper exists between ORM model and domain aggregate
- `reconstitute` used for loading from DB, not the public constructor
- Repository class declares `implements IXxxRepository`
- No business logic — pure I/O

**Presentation** (`src/*/presentation/`)
- Controllers validate input, call use case, map to response only
- Request/response DTOs separate from domain DTOs
- Domain exceptions mapped to HTTP status codes here, not in use case

## Severity tags

- **[BLOCKER]** — Layer violation, missing invariant, data loss, security issue
- **[BUG]** — Incorrect behavior, wrong type, unhandled null/undefined
- **[DESIGN]** — Anemic model, fat handler, missing domain event
- **[SIMPLIFY]** — Unnecessary complexity or duplication
- **[NIT]** — Minor style issue (only mention if it genuinely confuses readers)

## DDD-specific red flags (always call these out)

1. `customerId: string` in a domain method — should be `CustomerId` value object
2. One command modifying two separate aggregate roots
3. Application service calling another application service
4. Domain event handled in same transaction without an event bus
5. Loading from DB via the creation constructor (should use `reconstitute`)

## Output format

Numbered list, most critical first. Each finding: **[TAG] Title** → location + problem + fix.
