---
name: zod-express-validation
description: Runtime request validation with Zod for Express and Socket.IO. Generates TypeScript DTOs via z.infer. Blocks unvalidated data from reaching Use Cases.
---

# Zod Validation & Type Safety Expert

Ensures no `any` types or unvalidated data ever cross from the Presentation layer (Express / Socket.IO) into the Application layer.

## When to Activate

- Creating a new Express route or controller
- Adding a new Socket.IO event listener
- Defining a DTO for a Use Case
- Reviewing code where `req.body` is passed directly to a Use Case
- Replacing manually written interfaces with `z.infer`-generated types
- Standardising 400 error responses across the API

## Core Principles

1. **Never Trust the Client** — Every `req.body`, `req.query`, `req.params`, and Socket.IO payload must pass through a Zod schema.
2. **Single Source of Truth** — Use `z.infer<typeof Schema>` as the DTO type. Do not write a separate TypeScript interface.

## Installation

```bash
npm install zod
```

## Schema Definition

```typescript
// src/presentation/validations/register-user.schema.ts
import { z } from 'zod';

export const RegisterUserSchema = z.object({
  email: z.string().email({ message: 'Must be a valid email address' }),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Must contain at least one uppercase letter')
    .regex(/[0-9]/, 'Must contain at least one number'),
});

// The DTO is derived from the schema — no manual interface needed
export type RegisterUserDto = z.infer<typeof RegisterUserSchema>;
```

## Generic Express Validation Middleware

```typescript
// src/presentation/middleware/validate-request.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { ZodSchema, ZodError } from 'zod';

export function validateBody<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      res.status(400).json(formatZodError(result.error));
      return;
    }

    // Replace raw body with the validated + typed data
    req.body = result.data;
    next();
  };
}

export function validateQuery<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.query);
    if (!result.success) { res.status(400).json(formatZodError(result.error)); return; }
    req.query = result.data as any;
    next();
  };
}

function formatZodError(error: ZodError) {
  return {
    error: {
      code: 'validation_error',
      message: 'Request validation failed',
      details: error.errors.map((e) => ({
        field: e.path.join('.'),
        message: e.message,
        code: e.code,
      })),
    },
  };
}
```

## Express Route Usage

```typescript
// src/presentation/routes/auth.routes.ts
import { Router } from 'express';
import { validateBody } from '../middleware/validate-request.middleware';
import { RegisterUserSchema } from '../validations/register-user.schema';
import { registerUserController } from '../../main/container';

const router = Router();

router.post(
  '/register',
  validateBody(RegisterUserSchema),   // ← validates; returns 400 on failure
  (req, res) => registerUserController.handle(req, res),
);

export { router as authRoutes };
```

## Controller — Receives Only Validated Data

```typescript
// src/presentation/controllers/register-user.controller.ts
import { Request, Response } from 'express';
import { RegisterUserUseCase } from '../../application/use-cases/register-user.use-case';
import { RegisterUserDto } from '../validations/register-user.schema';

export class RegisterUserController {
  constructor(private readonly useCase: RegisterUserUseCase) {}

  async handle(req: Request, res: Response): Promise<void> {
    // req.body is already typed as RegisterUserDto — middleware has validated it
    const dto = req.body as RegisterUserDto;
    await this.useCase.execute(dto);
    res.status(201).json({ message: 'User registered.' });
  }
}
```

## Socket.IO Payload Validation

```typescript
// src/presentation/sockets/chat.gateway.ts
import { Socket } from 'socket.io';
import { z } from 'zod';

const SendMessageSchema = z.object({
  roomId: z.string().uuid(),
  content: z.string().min(1).max(1000),
});

export function registerChatHandlers(socket: Socket): void {
  socket.on('chat:send', (payload: unknown) => {
    const result = SendMessageSchema.safeParse(payload);

    if (!result.success) {
      // Emit error back to the sender; do not propagate
      socket.emit('chat:error', {
        code: 'validation_error',
        details: result.error.errors,
      });
      return;
    }

    // result.data is now strongly typed
    const { roomId, content } = result.data;
    // … call Use Case
  });
}
```

## Path Params and Query Strings

```typescript
// Validate route params (e.g., /users/:id)
const UserParamsSchema = z.object({
  id: z.string().uuid({ message: 'id must be a valid UUID' }),
});

// Validate query strings (e.g., /users?page=1&limit=20)
const PaginationSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

router.get(
  '/users/:id/orders',
  validateParams(UserParamsSchema),   // custom middleware following same pattern
  validateQuery(PaginationSchema),
  (req, res) => ordersController.handle(req, res),
);
```

## Common Zod Patterns Cheatsheet

```typescript
// Strings
z.string().email()
z.string().uuid()
z.string().url()
z.string().min(1).max(255)
z.string().regex(/pattern/)

// Numbers
z.number().int()
z.number().positive()
z.number().min(0).max(100)
z.coerce.number()           // coerces string "42" → 42 (useful for query params)

// Optionals / defaults
z.string().optional()       // string | undefined
z.string().nullable()       // string | null
z.string().default('active')

// Enums
z.enum(['admin', 'user', 'guest'])

// Objects & arrays
z.object({ name: z.string() }).strict()   // rejects extra keys
z.array(z.string()).min(1).max(10)

// Transforms (sanitize output)
z.string().trim().toLowerCase()

// Refinements (cross-field validation)
z.object({ password: z.string(), confirm: z.string() })
  .refine((d) => d.password === d.confirm, {
    message: 'Passwords do not match',
    path: ['confirm'],
  });
```

## Anti-Patterns

| ❌ Never Do | ✅ Instead |
|---|---|
| `useCase.execute(req.body)` | Parse `req.body` with Zod schema first |
| `const dto = req.body as RegisterUserDto` | Use `schema.parse()` or validated middleware |
| `socket.on('msg', (data: MyType) => ...)` | `schema.safeParse(data)` before casting |
| Writing a manual TS interface alongside a schema | Use `z.infer<typeof Schema>` |
| Catching ZodError inside the controller | Handle in middleware; pass only clean data to controller |
| Swallowing parse errors silently | Always return `400` with `result.error.errors` |
