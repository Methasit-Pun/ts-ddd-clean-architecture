---
name: ts-ddd-clean-architecture
description: Enforces Hexagonal Architecture, Domain-Driven Design (DDD), and Event-Driven workflows for Node.js using Express, Prisma, and Socket.IO.
---

# TypeScript DDD & Clean Architecture Expert

Strict Software Architect specializing in Hexagonal Architecture, Domain-Driven Design (DDD), and Event-Driven Architecture. Target stack: Node.js, Express, Socket.IO, and Prisma.

## When to Activate

- Creating a new feature end-to-end (Entity → Use Case → Controller)
- Designing or reviewing domain models, aggregates, or value objects
- Adding a new Use Case (command or query)
- Wiring up a new port (repository interface or publisher interface)
- Reviewing code for layer boundary violations (Prisma bleed, HTTP bleed)
- Mapping Prisma models to DDD Entities inside the Infrastructure layer
- Generating Domain Events for significant business actions

## Directory Structure

```
src/
├── domain/           # Pure DDD Core — zero external dependencies
│   ├── entities/
│   ├── value-objects/
│   ├── events/
│   └── exceptions/
├── application/      # Use Cases, Ports (interfaces), DTOs, Event Handlers
│   ├── use-cases/
│   ├── ports/
│   └── dtos/
├── infrastructure/   # Adapters: Prisma repos, Socket.IO publisher
│   ├── repositories/
│   └── publishers/
├── presentation/     # Express controllers, Socket.IO listeners
│   ├── controllers/
│   └── sockets/
└── main/             # DI container + server bootstrap
```

## Domain Layer

### Value Objects

```typescript
// src/domain/value-objects/email.vo.ts
export class Email {
  private constructor(private readonly value: string) {}

  static create(raw: string): Email {
    if (!raw.includes('@')) throw new InvalidEmailException(raw);
    return new Email(raw.toLowerCase().trim());
  }

  toString(): string { return this.value; }
  equals(other: Email): boolean { return this.value === other.value; }
}
```

### Entities / Aggregates

```typescript
// src/domain/entities/user.entity.ts
import { UserRegisteredEvent } from '../events/user-registered.event';

export class User {
  private readonly domainEvents: DomainEvent[] = [];

  private constructor(
    private readonly id: UserId,
    private readonly email: Email,
    private passwordHash: string,
  ) {}

  static register(id: UserId, email: Email, passwordHash: string): User {
    const user = new User(id, email, passwordHash);
    user.domainEvents.push(new UserRegisteredEvent(id, email));
    return user;
  }

  changePassword(newHash: string): void {
    if (!newHash) throw new WeakPasswordException();
    this.passwordHash = newHash;
  }

  pullDomainEvents(): DomainEvent[] {
    const events = [...this.domainEvents];
    this.domainEvents.length = 0;
    return events;
  }

  getId(): UserId { return this.id; }
  getEmail(): Email { return this.email; }
}
```

### Domain Events

```typescript
// src/domain/events/user-registered.event.ts
export class UserRegisteredEvent implements DomainEvent {
  readonly occurredOn: Date = new Date();
  constructor(
    readonly userId: UserId,
    readonly email: Email,
  ) {}
}
```

### Domain Exceptions

```typescript
// src/domain/exceptions/invalid-email.exception.ts
export class InvalidEmailException extends Error {
  constructor(raw: string) {
    super(`"${raw}" is not a valid email address.`);
    this.name = 'InvalidEmailException';
  }
}
```

## Application Layer

### Ports (Interfaces)

```typescript
// src/application/ports/user-repository.port.ts
export interface IUserRepository {
  findById(id: UserId): Promise<User | null>;
  findByEmail(email: Email): Promise<User | null>;
  save(user: User): Promise<void>;
}

// src/application/ports/realtime-publisher.port.ts
export interface IRealTimePublisher {
  publish(event: string, payload: unknown): void;
}
```

### Use Cases

```typescript
// src/application/use-cases/register-user.use-case.ts
export class RegisterUserUseCase {
  constructor(
    private readonly userRepo: IUserRepository,
    private readonly publisher: IRealTimePublisher,
  ) {}

  async execute(dto: RegisterUserDto): Promise<void> {
    const email = Email.create(dto.email);

    const existing = await this.userRepo.findByEmail(email);
    if (existing) throw new UserAlreadyExistsException();

    const id = UserId.generate();
    const hash = await hashPassword(dto.password);
    const user = User.register(id, email, hash);

    await this.userRepo.save(user);

    for (const event of user.pullDomainEvents()) {
      this.publisher.publish('user.registered', event);
    }
  }
}
```

## Infrastructure Layer

### Prisma Repository Adapter

```typescript
// src/infrastructure/repositories/prisma-user.repository.ts
export class PrismaUserRepository implements IUserRepository {
  constructor(private readonly db: PrismaClient) {}

  async findById(id: UserId): Promise<User | null> {
    const record = await this.db.user.findUnique({ where: { id: id.toString() } });
    if (!record) return null;
    return this.toDomain(record);       // ← always map; never leak Prisma types
  }

  async save(user: User): Promise<void> {
    await this.db.user.upsert({
      where: { id: user.getId().toString() },
      create: this.toPersistence(user),
      update: this.toPersistence(user),
    });
  }

  private toDomain(record: PrismaUser): User {
    return User.reconstitute(
      UserId.from(record.id),
      Email.create(record.email),
      record.passwordHash,
    );
  }

  private toPersistence(user: User) {
    return { id: user.getId().toString(), email: user.getEmail().toString() };
  }
}
```

## Presentation Layer

### Express Controller

```typescript
// src/presentation/controllers/register-user.controller.ts
export class RegisterUserController {
  constructor(private readonly useCase: RegisterUserUseCase) {}

  async handle(req: Request, res: Response): Promise<void> {
    const dto = RegisterUserSchema.parse(req.body); // Zod — see zod-express-validation skill
    await this.useCase.execute(dto);
    res.status(201).json({ message: 'User registered.' });
  }
}
```

## Dependency Injection (main/)

```typescript
// src/main/container.ts
const prisma = new PrismaClient();
const userRepo = new PrismaUserRepository(prisma);
const publisher = new SocketIOPublisher(io);
const registerUserUseCase = new RegisterUserUseCase(userRepo, publisher);
export const registerUserController = new RegisterUserController(registerUserUseCase);
```

## Step-by-Step Execution Workflow

When generating a new feature, work in this exact order:
1. **Domain** — Value Objects → Entity/Aggregate → Domain Events → Domain Exceptions
2. **Ports** — `IRepository` and `IRealTimePublisher` interfaces in `application/ports/`
3. **Use Case** — Orchestrates fetch → method call → save → publish
4. **Infrastructure** — Prisma Repository adapter with `toDomain` / `toPersistence` mappers
5. **Presentation** — Express Controller parses HTTP → DTO → calls Use Case
6. **DI** — Wire up in `main/container.ts`

## Strict Anti-Patterns

| ❌ Pattern | ✅ Correct Approach |
|---|---|
| `import { PrismaClient } from '@prisma/client'` in Domain/Application | Use the `IUserRepository` port interface |
| `useCase.execute(req.body)` | Parse via Zod first, pass typed DTO |
| `import { Server } from 'socket.io'` in Use Case | Call `IRealTimePublisher.publish()` |
| `prisma.user.findMany()` in Controller | Call Use Case, which calls Repository port |
| `user.password = newHash` | `user.changePassword(newHash)` — behavior, not setters |
