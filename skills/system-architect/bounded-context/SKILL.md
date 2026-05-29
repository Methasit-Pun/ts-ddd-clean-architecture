---
name: ts-ddd-bounded-context
description: Map bounded contexts for a TypeScript DDD project — identify context boundaries, define ubiquitous language, map relationships (ACL, Partnership, Shared Kernel, etc.), and surface aggregate roots and domain events. Trigger when the user says "map the domain", "define bounded contexts", "what are the aggregates", "design the domain model", "identify the ubiquitous language", "how should we split this domain", or when planning a new feature that touches multiple domain areas. Also trigger when the user describes a business process and wants to know how to structure it in DDD.
---

# Bounded Context Mapper — TypeScript DDD

Produce a structured domain map that gives the team a shared understanding of where responsibilities live, how contexts relate, and what language each context owns.

## Process (follow in order)

### 1. Extract domain concepts

From the user's description, identify and list:
- **Nouns** → candidate Entities and Value Objects
- **Verbs** → candidate Domain Events and Commands
- **Business rules** → candidate invariants for Aggregates

### 2. Group into bounded contexts

Cluster concepts that share a ubiquitous language and a clear owner. Each bounded context should:
- Have a single team or module owning it
- Have a name that domain experts recognize
- Be cohesive — the language inside it doesn't leak to other contexts

### 3. Define relationships

For each pair of interacting contexts, name the relationship pattern:

| Pattern | When to use |
|---------|-------------|
| **Shared Kernel** | Two contexts share a small, stable subset of the model |
| **Customer/Supplier** | One context consumes another's output; upstream is the supplier |
| **Conformist** | Downstream adapts entirely to upstream's model |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream's model to protect its own language |
| **Open Host Service** | Upstream publishes a stable, versioned API for many consumers |
| **Published Language** | Shared data format (e.g. events on a bus) both sides agree on |
| **Partnership** | Two contexts evolve together with joint planning |
| **Separate Ways** | No integration — contexts are fully independent |

### 4. Identify aggregates per context

For each context, name:
- The **Aggregate Root** (the entity that owns the transaction boundary)
- Its **children** (entities and value objects inside the boundary)
- The **domain events** it emits

### 5. Define ubiquitous language

For each context, list the key terms with one-line definitions. If the same word means different things in different contexts (e.g. "Order" in Billing vs. Shipping), call it out explicitly — this is a **linguistic boundary**.

---

## Output format

````markdown
# Domain Map: [System / Feature Name]

## Bounded Contexts

### [ContextName]
**Responsibility:** [one sentence]
**Owner:** [team or module name]
**Aggregate Roots:**
- `[AggregateName]` — [one-line responsibility]
  - Children: `[Entity]`, `[ValueObject]`
  - Emits: `[DomainEventName]`, ...

**Ubiquitous Language:**
| Term | Definition |
|------|-----------|
| [Term] | [Definition as understood inside this context only] |

---

### [AnotherContextName]
...

---

## Context Map

```
[ContextA] --[ACL]--> [ContextB]
[ContextB] --[Customer/Supplier]--> [ContextC]
[ContextA] --[Shared Kernel]--> [ContextD]
```

### Relationship details
- **[ContextA] → [ContextB]** (ACL): [ContextB] translates [ContextA]'s `Order` into its own `Purchase` to avoid coupling.
- **[ContextB] → [ContextC]** (Customer/Supplier): [ContextB] publishes `OrderShippedEvent`; [ContextC] subscribes.

---

## Linguistic boundaries

List any terms that appear in multiple contexts with *different meanings*. These are red flags for accidental coupling.

| Term | [ContextA] meaning | [ContextB] meaning |
|------|-------------------|-------------------|
| Order | Customer's purchase intent | Warehouse pick-list |

---

## Domain events flow

```
[AggregateA] --emits--> [DomainEventX] --consumed by--> [ContextB handler]
```

---

## Open questions

- [ ] [Unresolved design decision or missing domain knowledge]
````

---

## TypeScript folder structure (suggest this after mapping)

```
src/
  [context-name]/               ← one folder per bounded context
    domain/
      [AggregateName].ts        ← aggregate root
      [ValueObject].ts
      events/
        [DomainEventName].ts
      repositories/
        I[AggregateName]Repository.ts   ← interface only
    application/
      use-cases/
        [ActionName]UseCase.ts
      dtos/
    infrastructure/
      persistence/
        [AggregateName]Repository.ts   ← implementation
```

---

## Quality checks

Before finishing the map, verify:
- [ ] Every aggregate root has at least one domain event
- [ ] No entity is shared between two contexts without a Shared Kernel or ACL
- [ ] Every relationship has a named pattern
- [ ] Linguistic boundaries are documented
- [ ] The map answers: "who owns the data?" for every core entity
