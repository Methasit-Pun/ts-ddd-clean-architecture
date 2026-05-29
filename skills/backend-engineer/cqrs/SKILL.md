---
name: ts-ddd-cqrs
description: Design and implement the CQRS (Command Query Responsibility Segregation) pattern in a TypeScript DDD clean architecture project. Trigger when the user says "implement CQRS", "add a command", "add a query", "create a use case", "add a command handler", "build the application layer", "set up the command bus", or when the user needs to add a new feature and is asking how to wire up the application layer. Also trigger when distinguishing between write operations (commands) and read operations (queries) in any context.
---

# CQRS Pattern — TypeScript DDD

Implement the Command Query Responsibility Segregation pattern cleanly within the Application layer. Commands mutate state; queries read state. They never cross.

## Core rule

> A method is either a **Command** (changes state, returns void or the aggregate ID) or a **Query** (returns data, changes nothing). Never both.

---

## Folder structure

```
src/[context]/
  application/
    commands/
      [ActionName]/
        [ActionName]Command.ts       ← the DTO (plain data)
        [ActionName]CommandHandler.ts
    queries/
      [QueryName]/
        [QueryName]Query.ts
        [QueryName]QueryHandler.ts
        [QueryName]Result.ts         ← typed read model
    ports/
      ICommandBus.ts
      IQueryBus.ts
```

---

## Command implementation

### Command DTO
```typescript
// Commands are plain data bags — no methods, no logic
export class CreateOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly items: { productId: string; quantity: number }[],
  ) {}
}
```

### Command Handler
```typescript
// Handler lives in Application layer — it orchestrates, never contains business logic
export class CreateOrderCommandHandler {
  constructor(
    private readonly orderRepository: IOrderRepository,  // domain interface
    private readonly eventBus: IEventBus,
  ) {}

  async execute(command: CreateOrderCommand): Promise<string> {
    const order = Order.create(command.customerId, command.items); // domain logic here
    await this.orderRepository.save(order);
    await this.eventBus.publishAll(order.pullDomainEvents());
    return order.id.value;
  }
}
```

**Handler rules:**
- Handlers never contain `if/else` business logic — that belongs in the Aggregate
- Handlers are thin orchestrators: load → execute domain method → persist → publish events
- Return only the aggregate ID or `void` — never a rich domain object

---

## Query implementation

### Query DTO
```typescript
export class GetOrderByIdQuery {
  constructor(public readonly orderId: string) {}
}
```

### Read Model (Result)
```typescript
// Read models are flat, serializable, presentation-friendly
export interface OrderDetailResult {
  id: string;
  customerId: string;
  status: string;
  items: { productId: string; name: string; quantity: number; price: number }[];
  total: number;
  createdAt: string;
}
```

### Query Handler
```typescript
export class GetOrderByIdQueryHandler {
  constructor(private readonly orderReadRepository: IOrderReadRepository) {}

  async execute(query: GetOrderByIdQuery): Promise<OrderDetailResult | null> {
    return this.orderReadRepository.findById(query.orderId);
  }
}
```

**Query rules:**
- Query handlers talk to **read models**, not domain aggregates
- Read models can query the DB directly (views, joins, projections) — no aggregate loading
- Never raise domain events from a query handler

---

## Command/Query Bus interfaces

Define buses as ports in the application layer:

```typescript
// src/[context]/application/ports/ICommandBus.ts
export interface ICommandBus {
  execute<TResult>(command: object): Promise<TResult>;
}

// src/[context]/application/ports/IQueryBus.ts
export interface IQueryBus {
  ask<TResult>(query: object): Promise<TResult>;
}
```

Wire implementations in Infrastructure (e.g. NestJS `CqrsModule`, custom map, or a library like `@nestjs/cqrs`).

---

## Domain events after commands

After a command mutates an aggregate, collect and publish domain events:

```typescript
// Aggregate tracks events internally
class Order extends AggregateRoot {
  private _domainEvents: DomainEvent[] = [];

  static create(...): Order {
    const order = new Order(...);
    order._domainEvents.push(new OrderCreatedEvent(order.id.value));
    return order;
  }

  pullDomainEvents(): DomainEvent[] {
    const events = [...this._domainEvents];
    this._domainEvents = [];
    return events;
  }
}
```

---

## Common mistakes to avoid

| Mistake | Fix |
|---------|-----|
| Business logic in command handler | Move to aggregate method |
| Returning domain object from query | Return a flat read model DTO |
| Query handler loading an aggregate | Use a dedicated read repository |
| Command handler reading from two aggregates | One command = one aggregate transaction |
| Shared read/write repository interface | Split into `IOrderRepository` (write) and `IOrderReadRepository` (read) |

---

## When to use full CQRS vs. simple use cases

Use full CQRS (separate read/write models + buses) when:
- Query complexity justifies denormalized read models
- Read and write loads scale differently
- You need event sourcing on the write side

Use simple use cases (no bus, no separate read model) when:
- The context is small and queries are simple
- You want to avoid premature complexity

Document the choice in an ADR. See `system-architect/adr-writer`.
