---
name: docker-redis-iac
description: Multi-stage Docker builds, Docker Compose for local dev, Redis Socket.IO adapter for horizontal scaling, and safe Prisma production migrations.
---

# Infrastructure & Containerization Expert

DevOps and Infrastructure specialist for Node.js apps using Prisma, Express, and Socket.IO. Goal: stateless, scalable, production-ready containers.

## When to Activate

- Dockerizing a Node.js / Express application for the first time
- Setting up local dev environment with `docker-compose`
- Scaling WebSockets horizontally using Redis adapter
- Configuring safe `prisma migrate deploy` in CI/CD or Docker entrypoint
- Writing multi-stage Dockerfiles to minimise image size
- Adding PostgreSQL or Redis services to the stack
- Diagnosing "sessions lost on scale-out" or Socket.IO cross-instance issues

## Core Principles

1. **Stateless Node Servers** — No in-memory session or Socket.IO room state. Redis backs all shared state.
2. **Immutable Containers** — Code is baked in at build time; no live code mounts in production.
3. **Safe Migrations** — `prisma migrate deploy` in production only; `prisma db push` is banned.

## Multi-Stage Dockerfile

```dockerfile
# ── Stage 1: Builder ───────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci                          # reproducible install; respects package-lock.json

COPY prisma ./prisma
RUN npx prisma generate             # generate client before compile

COPY tsconfig.json .
COPY src ./src
RUN npm run build                   # outputs to /app/dist

# ── Stage 2: Runner (lean production image) ───────────────────
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev               # production deps only

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

EXPOSE 3000
CMD ["node", "dist/main/server.js"]
```

**Why two stages?** The builder needs TypeScript compiler, dev deps, and prisma CLI (~600 MB). The runner copies only compiled JS + prod deps (~80 MB).

## Docker Compose (Local Development)

```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build: .
    ports:
      - '3000:3000'
    environment:
      DATABASE_URL: postgres://dev:dev@postgres:5432/appdb
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: appdb
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U dev']
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'

volumes:
  postgres_data:
```

## Socket.IO Redis Adapter (Horizontal Scaling)

Install:
```bash
npm install @socket.io/redis-adapter ioredis
```

Wire up **only** in the Infrastructure bootstrap — the Application layer stays unaware of Redis:

```typescript
// src/infrastructure/realtime/socket-io.bootstrap.ts
import { createAdapter } from '@socket.io/redis-adapter';
import { Redis } from 'ioredis';
import { Server } from 'socket.io';

export function attachRedisAdapter(io: Server): void {
  const pubClient = new Redis(process.env.REDIS_URL!);
  const subClient = pubClient.duplicate();

  io.adapter(createAdapter(pubClient, subClient));
  console.log('[Socket.IO] Redis adapter attached');
}
```

```typescript
// src/main/server.ts
import { attachRedisAdapter } from '../infrastructure/realtime/socket-io.bootstrap';

const io = new Server(httpServer, { cors: { origin: '*' } });
attachRedisAdapter(io);  // ← Redis wired here, NOT in use cases
```

### IRealTimePublisher Stays Clean

```typescript
// src/infrastructure/publishers/socketio.publisher.ts
import { IRealTimePublisher } from '../../application/ports/realtime-publisher.port';
import { Server } from 'socket.io';

export class SocketIOPublisher implements IRealTimePublisher {
  constructor(private readonly io: Server) {}

  publish(event: string, payload: unknown): void {
    // io.emit broadcasts across ALL nodes via the Redis adapter automatically
    this.io.emit(event, payload);
  }
}
```

## Prisma Production Migrations

### Dockerfile Entrypoint (run migration then start)

```sh
# entrypoint.sh
#!/bin/sh
set -e
echo "Running database migrations..."
npx prisma migrate deploy         # safe, does NOT re-run applied migrations
echo "Starting server..."
exec node dist/main/server.js
```

```dockerfile
# In Dockerfile runner stage — replace CMD with entrypoint
COPY entrypoint.sh .
RUN chmod +x entrypoint.sh
ENTRYPOINT ["./entrypoint.sh"]
```

### CI/CD Pipeline Step (GitHub Actions example)

```yaml
- name: Run Prisma Migrations
  run: npx prisma migrate deploy
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

## Environment Variables Reference

| Variable | Example | Where set |
|---|---|---|
| `DATABASE_URL` | `postgres://user:pass@host:5432/db` | `.env`, Docker secret |
| `REDIS_URL` | `redis://redis:6379` | `.env`, Docker secret |
| `NODE_ENV` | `production` | Dockerfile `ENV` directive |
| `PORT` | `3000` | Optional, defaults to 3000 |

## Execution Workflow

When asked to dockerize or scale the application:
1. Check required environment variables (`DATABASE_URL`, `REDIS_URL`)
2. Write multi-stage `Dockerfile` (builder → runner)
3. Write or update `docker-compose.yml` for local dev (postgres + redis)
4. Add `entrypoint.sh` running `prisma migrate deploy` before server start
5. If multi-instance scaling needed: implement Redis adapter in `infrastructure/` layer only

## Anti-Patterns

| ❌ Never Do | ✅ Instead |
|---|---|
| `prisma db push` in production | `prisma migrate deploy` |
| Single-stage Dockerfile with dev deps | Multi-stage: builder + runner |
| Storing Socket.IO rooms in memory | Use `@socket.io/redis-adapter` |
| Importing `ioredis` in a Use Case | Inject `IRealTimePublisher` port |
| Hardcoding secrets in `docker-compose.yml` | Use environment variables / Docker secrets |
| `npm install` in runner stage | `npm ci --omit=dev` in dedicated stage |
