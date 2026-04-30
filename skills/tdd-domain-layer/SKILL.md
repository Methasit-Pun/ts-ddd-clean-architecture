---
name: tdd-domain-layer
description: Strict TDD for Domain and Application layers in Clean Architecture TypeScript. Red-Green-Refactor with Vitest. Mocks ports, never hits a database.
---

# TDD Domain & Application Expert

Strict Test-Driven Development practitioner for Clean Architecture TypeScript using **Vitest** (or Jest). Tests for Domain and Application layers must never connect to a database, external API, or Express server.

## When to Activate

- Adding a new business rule to an Entity or Aggregate
- Building a new Use Case (command or query)
- Fixing a bug in the Domain or Application layer
- Creating a new Value Object with validation rules
- Ensuring Domain Events are raised correctly
- Verifying that a Use Case calls the right repository methods
- Reviewing tests that are hitting the database (violation — fix them)

## Core Principles

1. **Red-Green-Refactor** — Write the failing test FIRST. Then write the minimum implementation to make it pass. Then refactor.
2. **Total Isolation** — Domain and Application tests must never import `@prisma/client`, `express`, or `socket.io`.
3. **Mock Ports, Not Domain** — In Use Case tests, mock repository and publisher interfaces. Never mock Domain Entities.

## Project Setup

```bash
npm install -D vitest @vitest/coverage-v8
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      include: ['src/domain/**', 'src/application/**'],
      thresholds: { lines: 80, functions: 80 },
    },
  },
});
```

## Domain Layer Tests — Entities & Value Objects

**Focus:** State transitions, invariant enforcement, Domain Event generation.

### Value Object Test

```typescript
// src/domain/value-objects/__tests__/email.vo.spec.ts
import { describe, it, expect } from 'vitest';
import { Email } from '../email.vo';
import { InvalidEmailException } from '../../exceptions/invalid-email.exception';

describe('Email Value Object', () => {
  it('creates a valid email in lowercase', () => {
    const email = Email.create('Alice@Example.COM');
    expect(email.toString()).toBe('alice@example.com');
  });

  it('trims whitespace', () => {
    const email = Email.create('  bob@example.com  ');
    expect(email.toString()).toBe('bob@example.com');
  });

  it('throws InvalidEmailException for an address without @', () => {
    expect(() => Email.create('notanemail')).toThrow(InvalidEmailException);
  });

  it('considers two emails with same value as equal', () => {
    const a = Email.create('alice@example.com');
    const b = Email.create('alice@example.com');
    expect(a.equals(b)).toBe(true);
  });
});
```

### Entity / Aggregate Test

```typescript
// src/domain/entities/__tests__/user.entity.spec.ts
import { describe, it, expect } from 'vitest';
import { User } from '../user.entity';
import { Email } from '../../value-objects/email.vo';
import { UserId } from '../../value-objects/user-id.vo';
import { UserRegisteredEvent } from '../../events/user-registered.event';
import { WeakPasswordException } from '../../exceptions/weak-password.exception';

describe('User Entity', () => {
  const id = UserId.generate();
  const email = Email.create('alice@example.com');

  describe('User.register()', () => {
    it('creates a User with the given email', () => {
      const user = User.register(id, email, 'hashedPw123');
      expect(user.getEmail().toString()).toBe('alice@example.com');
    });

    it('emits a UserRegisteredEvent', () => {
      const user = User.register(id, email, 'hashedPw123');
      const events = user.pullDomainEvents();
      expect(events).toHaveLength(1);
      expect(events[0]).toBeInstanceOf(UserRegisteredEvent);
    });

    it('clears domain events after pulling them', () => {
      const user = User.register(id, email, 'hashedPw123');
      user.pullDomainEvents();
      expect(user.pullDomainEvents()).toHaveLength(0);
    });
  });

  describe('user.changePassword()', () => {
    it('updates the password hash', () => {
      const user = User.register(id, email, 'oldHash');
      user.changePassword('newStrongHash');
      // verify via a query method or reconstitution if needed
    });

    it('throws WeakPasswordException for an empty string', () => {
      const user = User.register(id, email, 'oldHash');
      expect(() => user.changePassword('')).toThrow(WeakPasswordException);
    });
  });
});
```

## Application Layer Tests — Use Cases

**Focus:** Orchestration. Does the Use Case fetch data, call the right Entity method, save it, and publish events?

### Mocking Ports

```typescript
// src/application/use-cases/__tests__/register-user.use-case.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { RegisterUserUseCase } from '../register-user.use-case';
import { IUserRepository } from '../../ports/user-repository.port';
import { IRealTimePublisher } from '../../ports/realtime-publisher.port';
import { UserAlreadyExistsException } from '../../../domain/exceptions/user-already-exists.exception';

const makeUserRepo = (): IUserRepository => ({
  findById: vi.fn(),
  findByEmail: vi.fn().mockResolvedValue(null),  // default: user does not exist
  save: vi.fn().mockResolvedValue(undefined),
});

const makePublisher = (): IRealTimePublisher => ({
  publish: vi.fn(),
});

describe('RegisterUserUseCase', () => {
  let userRepo: IUserRepository;
  let publisher: IRealTimePublisher;
  let useCase: RegisterUserUseCase;

  beforeEach(() => {
    userRepo = makeUserRepo();
    publisher = makePublisher();
    useCase = new RegisterUserUseCase(userRepo, publisher);
  });

  it('saves the new user to the repository', async () => {
    await useCase.execute({ email: 'alice@example.com', password: 'StrongPass1!' });
    expect(userRepo.save).toHaveBeenCalledOnce();
  });

  it('publishes a user.registered event', async () => {
    await useCase.execute({ email: 'alice@example.com', password: 'StrongPass1!' });
    expect(publisher.publish).toHaveBeenCalledWith('user.registered', expect.objectContaining({
      email: expect.objectContaining({}),
    }));
  });

  it('throws UserAlreadyExistsException when email is taken', async () => {
    vi.mocked(userRepo.findByEmail).mockResolvedValue({ getId: vi.fn() } as any);

    await expect(
      useCase.execute({ email: 'existing@example.com', password: 'StrongPass1!' }),
    ).rejects.toThrow(UserAlreadyExistsException);

    expect(userRepo.save).not.toHaveBeenCalled();
  });

  it('does NOT call the publisher if saving the user fails', async () => {
    vi.mocked(userRepo.save).mockRejectedValue(new Error('DB error'));

    await expect(
      useCase.execute({ email: 'alice@example.com', password: 'StrongPass1!' }),
    ).rejects.toThrow('DB error');

    expect(publisher.publish).not.toHaveBeenCalled();
  });
});
```

## Execution Workflow (Red-Green-Refactor)

```
1. Write .spec.ts — describe expected behaviour, all tests FAIL (red)
2. Run: npx vitest run
3. Write minimum implementation to make tests pass (green)
4. Run: npx vitest run  ← must all pass
5. Refactor implementation (no behaviour change)
6. Run: npx vitest run  ← still all passing
```

## Useful NPM Scripts

```jsonc
// package.json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

## Anti-Patterns

| ❌ Never Do | ✅ Instead |
|---|---|
| `import { PrismaClient } from '@prisma/client'` in a `.spec.ts` | Mock the `IRepository` interface with `vi.fn()` |
| `import supertest from 'supertest'` in a Domain/Use Case test | Use `supertest` only in E2E / integration tests |
| `jest.mock('../user.entity')` | Never mock Domain Entities — test them directly |
| Testing implementation details (`user._passwordHash`) | Test behaviour via public methods |
| Writing implementation before the test | Write the failing test first, always |
| Single giant test covering everything | One `it()` per behaviour, descriptive names |
