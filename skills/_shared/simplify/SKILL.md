---
name: ts-ddd-simplify
description: Review and simplify TypeScript DDD code — remove unnecessary complexity, eliminate duplication, reduce premature abstraction. Trigger when the user says "simplify this", "too much boilerplate", "over-engineered", "clean this up", "refactor this", or when code has grown too abstract or introduces patterns before they prove value.
---

# Simplify — TypeScript DDD

> Three similar lines is better than a premature abstraction. Abstract only when a pattern has appeared in at least three real cases.

## What to look for

**Domain over-engineering**
- VO wrapping a single `string` with no validation or behavior — remove it (keep if it enforces format like `Email`, `Money`)
- Aggregate that only delegates to a domain service for every operation — collapse the service into the aggregate
- Deeply nested event hierarchy (`DomainEvent → OrderEvent → OrderCancelledEvent`) — flatten unless used for polymorphic handling

**Application over-engineering**
- Handler with one line `return this.repo.findById(id)` — skip the use case, call the repo from the query handler directly
- Command DTO with fields identical to the domain constructor params — collapse into one
- Interface with a single implementation that is never mocked in tests — inject the concrete class; add the interface when a second impl is needed

**Infrastructure over-engineering**
- Mapper that copies fields one-for-one with no type conversion — consider if domain and ORM models are actually the same thing at this stage
- Repository wrapping ORM with one-liner methods and no mapping or error handling — the layer adds no value yet

## Premature patterns

| Pattern | Worth it when | Premature when |
|---------|---------------|----------------|
| CQRS with separate read models | Queries are complex joins or aggregations | Simple CRUD reads |
| Event sourcing | Full audit trail or temporal queries required | Standard transactional state |
| Outbox pattern | Distributed at-least-once delivery needed | Single service, no message broker |
| Sagas / Process Managers | Multi-step multi-service long-running workflows | Single aggregate transitions |

## What NOT to simplify

- Invariant-enforcing VOs even if they look like "just a string wrapper"
- Repository interface when it enables test doubles
- Domain events even if only one subscriber currently listens
- Mapper even if trivial — it protects the domain from ORM schema changes

## Output format

Numbered list. Each finding: **[REMOVE]** / **[COLLAPSE]** / **[DEFER]** + file path + why it's safe to remove + what to do instead.
