---
name: ts-ddd-adr-writer
description: Write a structured Architecture Decision Record (ADR) for any technology or design choice in a TypeScript DDD clean architecture project. Trigger when the user says "document this decision", "write an ADR", "why did we choose X", "we decided to use Y instead of Z", "record this architecture choice", or when a significant design trade-off is being discussed and should be captured for the team. Also trigger when the user debates between two patterns (e.g. TypeORM vs Prisma, REST vs gRPC, monolith vs microservices).
---

# ADR Writer — TypeScript DDD

Produce a clear, concise Architecture Decision Record that captures the context, trade-offs, and outcome of a design decision. Good ADRs are written *at decision time*, not retrospectively — they record what was known and what was uncertain when the decision was made.

## ADR file location

Save ADRs to `docs/adr/` using the naming convention:
```
docs/adr/NNNN-short-title-in-kebab-case.md
```
Increment `NNNN` from the highest existing number. If no ADRs exist yet, start at `0001`.

---

## ADR template

Use this exact structure:

```markdown
# ADR-NNNN: [Title — one sentence, imperative mood]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXXX
**Deciders:** [roles or names, e.g. "System Architect, Backend Lead"]
**Tags:** [domain, infrastructure, testing, security, etc.]

---

## Context

[2-4 sentences. What problem or constraint forced this decision? What is the current state of the system? What are the forces at play — technical constraints, team skills, time pressure, compliance requirements?]

## Decision Drivers

- [Most important criterion — e.g. "must not couple domain layer to ORM"]
- [Second criterion]
- [Third criterion]

## Considered Options

| Option | Summary |
|--------|---------|
| **Option A** | [one-line description] |
| **Option B** | [one-line description] |
| **Option C** | [one-line description, if applicable] |

## Decision

**We chose Option [X].**

[1-2 sentences explaining the primary reason.]

## Consequences

### Positive
- [concrete benefit this decision enables]
- [another benefit]

### Negative / Trade-offs
- [concrete cost or constraint introduced]
- [another trade-off]

### Neutral
- [things that change but are neither good nor bad]

## Option Analysis

### Option A — [Name]
**Pros:** ...
**Cons:** ...

### Option B — [Name]
**Pros:** ...
**Cons:** ...

## Implementation Notes

[Optional. Specific things the team must do or avoid when implementing this decision. Reference specific files or patterns if relevant.]

## Links

- [Related ADR, RFC, or ticket]
- [External reference, doc, or benchmark]
```

---

## Quality rules

- **Status is always set.** Default to `Proposed` if the decision hasn't been ratified yet.
- **Context explains the *why*, not the *what*.** Avoid restating the decision in the context section.
- **Decision drivers are ranked by importance** — the most critical constraint first.
- **Consequences are honest.** Every decision has trade-offs. If the "Negative" section is empty, dig deeper.
- **Implementation notes are optional** — only include them if there's something non-obvious the reader needs to know to implement the decision correctly.
- **Tags enable filtering** — always tag with the layer (domain, application, infrastructure, presentation) and concern (orm, auth, testing, observability, etc.)

---

## DDD-specific context prompts

When writing ADRs for DDD decisions, consider surfacing these forces:

- Does this decision affect the **dependency rule**? (Does it risk infrastructure concerns leaking into domain?)
- Does it change the **ubiquitous language**? (Does a library force different naming than the domain model?)
- Does it affect **aggregate boundaries**? (Does the choice constrain how we model consistency boundaries?)
- Does it introduce a **shared kernel** or **anti-corruption layer** requirement?

---

## Example titles (good)

- `Use Prisma as the ORM for infrastructure layer implementations`
- `Adopt CQRS for read-heavy bounded contexts only`
- `Enforce one aggregate per transaction as the consistency boundary`
- `Use Domain Events over direct service calls for cross-context communication`

## Example titles (bad — too vague)

- `Database choice`
- `We picked Prisma`
- `Architecture update`
