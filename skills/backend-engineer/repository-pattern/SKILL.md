---
name: ts-ddd-repository-pattern
description: Design and implement the Repository pattern in a TypeScript DDD clean architecture project — define repository interfaces in the domain layer and implementations in the infrastructure layer. Trigger when the user says "create a repository", "implement persistence", "add database access", "wire up TypeORM/Prisma/Drizzle", "implement the repository interface", "add Unit of Work", "how do I persist this aggregate", or when connecting domain aggregates to any data store. Also trigger when reviewing persistence code that may be violating the dependency rule.
---

# Repository Pattern — TypeScript DDD

Repositories are the bridge between the domain and persistence. The **interface** lives in the domain layer; the **implementation** lives in infrastructure. This is the dependency inversion principle in action — domain depends on nothing outside itself.

## Core rule

> The domain defines *what* it needs. Infrastructure decides *how* to deliver it.

---

## Folder structure

```
src/[context]/
  domain/
    repositories/
      I[AggregateName]Repository.ts    ← interface (domain owns this)
  infrastructure/
    persistence/
      [AggregateName]Repository.ts     ← concrete implementation
      [AggregateName]OrmModel.ts       ← ORM/DB mapping model (infrastructure only)
      mappers/
        [AggregateName]Mapper.ts       ← maps between ORM model ↔ domain aggregate
```

---

## Domain repository interface

Keep it minimal — only the operations the domain actually needs:

```typescript
// src/[context]/domain/repositories/IOrderRepository.ts
export interface IOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
}
```

**Interface rules:**
- Parameters and return types use **domain types only** (Aggregates, Value Objects, Domain IDs)
- No ORM types, no database primitives, no `Promise<any>`
- Method names reflect the ubiquitous language, not SQL (`findById`, not `selectByPrimaryKey`)
- `save` handles both insert and update (upsert semantics) — callers don't think about DB state

---

## ORM model (infrastructure only)

The ORM model is a plain persistence concern — it never crosses into domain:

```typescript
// src/[context]/infrastructure/persistence/OrderOrmModel.ts
// Example with TypeORM
@Entity('orders')
export class OrderOrmModel {
  @PrimaryColumn('uuid') id: string;
  @Column() customerId: string;
  @Column() status: string;
  @CreateDateColumn() createdAt: Date;
  @OneToMany(() => OrderItemOrmModel, item => item.order, { cascade: true, eager: true })
  items: OrderItemOrmModel[];
}
```

---

## Mapper

The mapper translates between the persistence model and the domain aggregate. It lives in infrastructure and knows about both sides:

```typescript
// src/[context]/infrastructure/persistence/mappers/OrderMapper.ts
export class OrderMapper {
  static toDomain(raw: OrderOrmModel): Order {
    return Order.reconstitute({      // use a factory method, not the constructor
      id: new OrderId(raw.id),
      customerId: new CustomerId(raw.customerId),
      status: OrderStatus.fromString(raw.status),
      items: raw.items.map(OrderItemMapper.toDomain),
      createdAt: raw.createdAt,
    });
  }

  static toPersistence(order: Order): OrderOrmModel {
    const model = new OrderOrmModel();
    model.id = order.id.value;
    model.customerId = order.customerId.value;
    model.status = order.status.value;
    model.items = order.items.map(OrderItemMapper.toPersistence);
    return model;
  }
}
```

**Mapper rules:**
- Aggregates should expose a `reconstitute` static factory that bypasses constructor invariants (we trust DB data)
- Never call `new Order()` with raw DB strings — always go through the mapper
- The mapper is the only place that knows the ORM model schema

---

## Repository implementation

```typescript
// src/[context]/infrastructure/persistence/OrderRepository.ts
@Injectable()
export class OrderRepository implements IOrderRepository {
  constructor(
    @InjectRepository(OrderOrmModel)
    private readonly ormRepo: TypeOrmRepository<OrderOrmModel>,
  ) {}

  async findById(id: OrderId): Promise<Order | null> {
    const model = await this.ormRepo.findOne({ where: { id: id.value } });
    return model ? OrderMapper.toDomain(model) : null;
  }

  async save(order: Order): Promise<void> {
    const model = OrderMapper.toPersistence(order);
    await this.ormRepo.save(model);
  }

  async delete(id: OrderId): Promise<void> {
    await this.ormRepo.delete({ id: id.value });
  }
}
```

---

## Unit of Work (when needed)

Use Unit of Work when a command must persist changes to multiple aggregates atomically:

```typescript
// src/shared/application/ports/IUnitOfWork.ts
export interface IUnitOfWork {
  begin(): Promise<void>;
  commit(): Promise<void>;
  rollback(): Promise<void>;
  getRepository<T>(token: symbol): T;
}
```

Only introduce Unit of Work when you have a real multi-aggregate transaction need. Adding it preemptively is over-engineering.

---

## Prisma variant

If using Prisma instead of TypeORM, the same pattern applies — just different infrastructure wiring:

```typescript
export class OrderRepository implements IOrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const raw = await this.prisma.order.findUnique({
      where: { id: id.value },
      include: { items: true },
    });
    return raw ? OrderMapper.toDomain(raw) : null;
  }

  async save(order: Order): Promise<void> {
    const data = OrderMapper.toPrismaCreate(order);
    await this.prisma.order.upsert({
      where: { id: order.id.value },
      create: data,
      update: data,
    });
  }
}
```

---

## Read repository (for CQRS queries)

Separate the write repository from the read repository — queries should never load full aggregates:

```typescript
// src/[context]/domain/repositories/IOrderReadRepository.ts
export interface IOrderReadRepository {
  findById(id: string): Promise<OrderDetailResult | null>;
  findByCustomer(customerId: string): Promise<OrderSummaryResult[]>;
}
```

The read repository can return plain objects/interfaces — not domain aggregates. See `backend-engineer/cqrs` for full CQRS guidance.

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| ORM model used as domain entity | Create a separate domain aggregate and use a mapper |
| Repository contains business logic | Move invariant checks into the aggregate |
| Interface accepts ORM types as params | Replace with domain Value Objects |
| `findAll()` with no pagination | Add `findMany(criteria, pagination)` instead |
| Repository loads child aggregates of another aggregate | Each aggregate has its own repository — never load cross-boundary |
