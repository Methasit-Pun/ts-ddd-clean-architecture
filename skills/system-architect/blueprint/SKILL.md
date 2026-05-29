---
name: ts-ddd-blueprint
description: Turn a one-line feature or system objective into a complete, step-by-step construction plan for a TypeScript DDD clean architecture project. Each step is self-contained so a fresh agent can execute it cold. Use this skill whenever the user says "plan", "blueprint", "roadmap", "how do we build", "design the system for", or describes a feature that spans multiple layers (domain → application → infrastructure → presentation). Also trigger when the user asks to break down a bounded context, design an aggregate, or plan a new module.
---

# TypeScript DDD Blueprint

Turn a one-line objective into an executable, layered construction plan that respects the Clean Architecture dependency rule: **Domain ← Application ← Infrastructure ← Presentation**.

## Your job

Given a feature or system goal, produce a plan that:
- Identifies which **bounded contexts** are touched
- Breaks work into numbered steps, each owning exactly one layer concern
- Marks steps that can run in parallel
- Calls out the anti-patterns most likely to appear in this type of work
- Ends with a definition-of-done checklist

Keep the plan actionable. A future agent reading a step should know *exactly* what file to create, what interface to implement, and what invariant to enforce — without reading the other steps.

---

## Step format

Use this template for every step:

```
### Step N — [Layer] [Short title]
**Context:** One sentence on why this step exists and what depends on it.
**Files to create/modify:** List specific paths relative to `src/`
**Contract:** The TypeScript interface or type signature this step must satisfy
**Invariants:** Business rules or constraints that must hold after this step
**Done when:** Concrete, verifiable acceptance criteria
**Can parallelize with:** Step numbers (or "none")
```

---

## Layer order (always follow this sequence)

1. **Domain** — Entities, Value Objects, Aggregates, Domain Events, Repository *interfaces*, Domain Services
2. **Application** — Use Cases / Commands / Queries, DTOs, Application Services, Ports (abstract interfaces)
3. **Infrastructure** — Repository *implementations*, ORM models, external adapters, DI wiring
4. **Presentation** — Controllers, resolvers, CLI handlers, request/response schemas

Never let a lower layer import from a higher one. Flag it in the plan if the task would force that.

---

## Bounded context identification

Before writing steps, answer these questions:
- What is the core domain concept? (the noun that owns the behaviour)
- What are the supporting sub-domains?
- Are there any shared kernels or anti-corruption layers needed?

Name these explicitly at the top of the plan under `## Bounded Contexts`.

---

## Parallel step detection

After sequencing all steps, scan for steps that:
- Write to different bounded contexts with no shared state
- Operate on different layers with no compile-time dependency
- Can be tested in isolation

Mark those with `⚡ parallel` so teams can split work.

---

## Anti-pattern catalog

Call out the top 3 anti-patterns most likely to appear in this specific plan. Choose from:

- Anemic domain model (logic leaking into application/infrastructure)
- Fat aggregate (aggregate doing too much, violating SRP)
- Leaky abstraction (infrastructure types bleeding into domain)
- God use case (one use case orchestrating 5+ aggregates)
- Premature CQRS (adding read/write split before query complexity justifies it)
- Missing domain event (state change with no event = invisible side-effect)
- DTO mapped in wrong layer (mapping inside domain instead of application)

---

## Output format

```
# Blueprint: [Objective]

## Bounded Contexts
- **[ContextName]**: [one-line responsibility]

## Dependency graph
[ASCII or text diagram showing step dependencies]

## Steps
[All steps using the step format above]

## Anti-patterns to watch
1. ...
2. ...
3. ...

## Definition of Done
- [ ] All domain types are framework-free (no ORM/HTTP imports in src/domain/)
- [ ] Every aggregate has at least one domain event
- [ ] Repository interfaces live in domain/, implementations in infrastructure/
- [ ] Use cases have unit tests with mocked repositories
- [ ] No circular imports between bounded contexts
```

---

## Example

**Input:** "Add order cancellation to the e-commerce system"

**Produces:**
- Bounded context: `Orders`
- Step 1 (Domain): Add `CancelOrder` method to `Order` aggregate + `OrderCancelledEvent`
- Step 2 (Domain): Define `OrderRepository.findById` and `save` interface
- Step 3 (Application): `CancelOrderUseCase` command + handler
- Step 4 (Infrastructure): TypeORM `OrderRepositoryImpl`
- Step 5 (Presentation): `DELETE /orders/:id` controller
- Steps 2 & 4 can parallelize ⚡

---

## What NOT to include in the plan

- Implementation code (pseudocode only if strictly necessary for clarity)
- Framework-specific boilerplate that can be generated
- Steps for things already done (check `src/` first before adding a step)
