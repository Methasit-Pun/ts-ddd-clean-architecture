---
name: ts-ddd-simplify
description: Review and simplify TypeScript DDD clean architecture code — remove unnecessary complexity, eliminate duplication, reduce abstraction altitude, and clean up over-engineered patterns. Trigger when the user says "simplify this", "this feels over-engineered", "too much boilerplate", "can we reduce this", "clean this up", "this is too complex", "refactor this", or when code has grown too abstract, has too many layers of indirection, or introduces patterns before their value is proven. Also trigger when reviewing code that might have premature CQRS, unnecessary abstractions, or verbose value objects.
---

# Simplify — TypeScript DDD

Remove friction without removing correctness. The goal is code that is easier to understand and change — not shorter code for its own sake.

## Guiding principle

> Three similar lines is better than a premature abstraction. Only abstract when the pattern has proven itself in at least three real cases.

---

## What to look for

### Over-engineering in the domain layer

**Unnecessary Value Objects**
- A `Name` VO wrapping a single `string` with no validation or behavior is just boilerplate. Keep it if it enforces format (e.g. `Email`, `PhoneNumber`, `Money`). Remove it if it adds no invariant.

**Anemic aggregate with delegation**
- An aggregate that only delegates to a domain service for every operation is not an aggregate — it's a bag of getters. Consider collapsing the service into the aggregate or simplifying to a plain entity.

**Over-nested event hierarchies**
- `DomainEvent → OrderEvent → OrderStatusChangedEvent → OrderCancelledEvent` — flatten unless the hierarchy is actually used for polymorphic handling.

---

### Over-engineering in the application layer

**Handler that only calls one repository method**
```typescript
// Before — unnecessary use case
class GetOrderByIdUseCase {
  execute(id: string) { return this.repo.findById(id); }
}

// After — just expose the read repository via the query handler directly
// Only add a use case when there is orchestration logic
```

**DTO that mirrors the domain object exactly**
- If a command DTO has the same fields as the aggregate constructor with no transformation, it may not need to exist as a separate class.

**Interfaces with one implementation**
- `IEmailService` with only `SmtpEmailService` as its implementation adds indirection with no current benefit. Keep the interface if you genuinely expect a second implementation (test mock, provider swap). Remove it otherwise.

---

### Over-engineering in the infrastructure layer

**Mapper with no real transformation**
- If the mapper just copies field for field with no type conversion, consider whether the ORM model and domain model are actually the same thing at this stage, and whether the separation is premature.

**Repository wrapping the ORM with one-liner methods**
- If `findById` just calls `this.ormRepo.findOne(...)` with no mapping, data shaping, or error handling, the repository may be adding a layer with no value yet.

---

### Premature patterns

| Pattern | When it pays off | When it's premature |
|---------|-----------------|-------------------|
| CQRS with separate read models | Queries are complex joins or aggregations | CRUD with simple reads |
| Event sourcing | Full audit trail required, temporal queries needed | Standard transactional state |
| Outbox pattern | Distributed system with at-least-once delivery needs | Single service, no message broker |
| Sagas / Process Managers | Multi-step, multi-service long-running workflows | Single aggregate transitions |

---

## Simplification techniques

### 1. Collapse unnecessary indirection

Before:
```typescript
class OrderService {
  cancel(id: OrderId) { return this.aggregate.cancel(); }
}
// Handler calls OrderService which calls Aggregate

// After: Handler calls Aggregate directly
const order = await this.repo.findById(id);
order.cancel();
await this.repo.save(order);
```

### 2. Replace interface + single-impl with concrete class + extract-if-needed-later

Before: `IOrderRepository` → `OrderRepository` (only ever one impl)

After: Just `OrderRepository`. Add the interface when the second impl is needed (e.g. `InMemoryOrderRepository` for testing).

### 3. Merge thin use cases

If two use cases share all logic except one field, extract the shared logic as a domain method and have one use case call it.

### 4. Remove redundant DTOs

If `CreateOrderRequest` → `CreateOrderCommand` → `Order.create()` all have identical fields, collapse the request directly into the command.

---

## What NOT to simplify

- Invariant-enforcing Value Objects even if "just a string wrapper"
- Repository interface when it enables easy test doubles
- Domain events even if only one subscriber currently listens
- Mapper even if it seems trivial — it protects the domain from ORM schema changes
- Separation of read/write repositories once the read queries have diverged

---

## Output format

Report as a list. Each item: what to remove/collapse, why it's safe, and the simplified version or a pointer to it.

```
## Simplification Findings

### [REMOVE] IEmailService interface — single concrete implementation
`src/notifications/application/ports/IEmailService.ts` has one implementation
(SmtpEmailService) and is never mocked in tests. The interface adds a level of
indirection with no current benefit.
Suggestion: Delete the interface; inject SmtpEmailService directly. Extract the
interface later if a test double or a second provider is added.

### [COLLAPSE] GetUserProfileUseCase delegates entirely to UserReadRepository
The use case is one line: `return this.userReadRepository.findById(id)`.
Suggestion: Call the read repository directly from the query handler.
```
