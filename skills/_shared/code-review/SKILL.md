---
name: ts-ddd-code-review
description: Review TypeScript DDD clean architecture code for correctness bugs, layer violations, and simplification opportunities. Trigger when the user says "review this", "check this code", "code review", "what's wrong with this", "is this correct DDD?", "does this follow clean architecture?", or when changes touch domain, application, or infrastructure layers and should be checked for dependency rule violations. Also trigger when the user pastes code and asks for feedback, or before merging a PR in a DDD project.
---

# Code Review — TypeScript DDD

Review code at the right altitude. Focus on what matters: correctness, layer discipline, and DDD fidelity. Avoid style nits that linters already catch.

## Review checklist by layer

### Domain layer (`src/*/domain/`)

- [ ] **No framework imports** — domain files must not import from NestJS, TypeORM, Prisma, Express, or any infrastructure library
- [ ] **Aggregates enforce invariants** — business rules validated in the aggregate, not in handlers or services
- [ ] **Value Objects are immutable** — no setters, all properties `readonly`
- [ ] **Domain events are raised** — every meaningful state change emits a domain event
- [ ] **Repository interfaces use domain types** — no ORM entities or raw strings in interface signatures
- [ ] **Anemic domain model check** — if the aggregate has only getters and no behavior methods, flag it

### Application layer (`src/*/application/`)

- [ ] **Handlers are thin** — command/query handlers orchestrate, not decide
- [ ] **No domain logic in handlers** — `if (order.status === 'PENDING')` in a handler is a red flag
- [ ] **Commands return void or ID only** — never a domain aggregate
- [ ] **Queries use read models** — not domain aggregates
- [ ] **No infrastructure imports** — handlers depend on interfaces, not concrete classes

### Infrastructure layer (`src/*/infrastructure/`)

- [ ] **Mapper exists** — no direct `new DomainAggregate()` with ORM fields
- [ ] **`reconstitute` used for loading** — not the public constructor
- [ ] **Repository implements domain interface** — class declaration shows `implements IXxxRepository`
- [ ] **No business logic** — persistence implementations are pure I/O

### Presentation layer (`src/*/presentation/`)

- [ ] **No domain logic** — controllers validate input, call use case, map to response
- [ ] **Request/response DTOs are separate** from domain DTOs
- [ ] **Error mapping** — domain exceptions are translated to HTTP status codes here, not in the use case

---

## Layer dependency rule (most critical check)

Run this mental import graph check:

```
Domain    → imports nothing outside domain/
Application → imports domain only
Infrastructure → imports domain + application interfaces
Presentation → imports application only (never domain directly)
```

If any import violates this, **it is a blocker**, not a nit.

---

## Severity levels

Use these tags when reporting findings:

- **[BLOCKER]** — Layer violation, missing invariant, data loss risk, security issue
- **[BUG]** — Incorrect behavior, wrong return type, unhandled null/undefined
- **[DESIGN]** — Structural problem (anemic model, fat handler, missing domain event)
- **[SIMPLIFY]** — Unnecessary complexity, duplicated logic, premature abstraction
- **[NIT]** — Minor style issue (only mention if it genuinely confuses readers)

---

## Review format

Report findings as a numbered list. Lead with the most critical:

```
## Code Review

### [BLOCKER] Domain imports TypeORM entity
`src/orders/domain/Order.ts` line 3 imports `OrderOrmModel` from infrastructure.
This couples the domain to the persistence framework and breaks the dependency rule.
**Fix:** Remove the import. The ORM model belongs in `infrastructure/persistence/`.

### [BUG] findById returns wrong type
`OrderRepository.findById` returns `Order | undefined` but the interface declares
`Promise<Order | null>`. This causes a runtime null check mismatch in the handler.
**Fix:** Return `null` explicitly when the entity is not found.

### [DESIGN] No domain event on state change
`Order.cancel()` mutates `this.status` but does not emit `OrderCancelledEvent`.
Downstream contexts (e.g. Billing, Notifications) will miss this transition.
**Fix:** Add `this._domainEvents.push(new OrderCancelledEvent(this.id.value))`.
```

---

## What NOT to flag

- Formatting, semicolons, quote style — linters handle these
- Variable naming unless it violates ubiquitous language
- Performance optimizations without a measured bottleneck
- Abstractions that work fine for the current scale
- Personal preference on code style not related to correctness or DDD principles

---

## DDD-specific red flags (always call these out)

1. **Primitive obsession in domain** — `customerId: string` in a domain method instead of `CustomerId` value object
2. **Aggregate crossing transaction boundary** — one command modifying two different aggregate roots
3. **Service calling service** — application services calling other application services (creates implicit coupling)
4. **Domain event handled in same transaction without event bus** — side effects in the aggregate itself
5. **Missing `reconstitute` factory** — loading from DB using the same constructor as creation (bypasses valid invariant check timing)
