---
name: ts-ddd-ci-design
description: Design and implement CI/CD pipelines for a TypeScript DDD clean architecture project — GitHub Actions, GitLab CI, Docker builds, environment promotion, and secrets management. Trigger when the user says "set up CI", "add a pipeline", "automate tests", "write a GitHub Actions workflow", "configure deployment", "add Docker support", "set up CD", "automate the build", or when the project needs automated quality gates before merge. Also trigger when the user asks about environment promotion (dev → staging → prod) or secrets management strategy.
---

# CI/CD Pipeline Design — TypeScript DDD

Design pipelines that enforce quality gates, protect the dependency rule, and promote artifacts safely from development through to production.

## Pipeline philosophy

A good pipeline for a DDD project enforces:
1. **Type safety** — `tsc --noEmit` catches boundary violations at compile time
2. **Layer isolation** — import linting (e.g. `dependency-cruiser`) prevents domain from importing infrastructure
3. **Test pyramid** — unit tests run first (fast), integration tests run second, E2E last
4. **One artifact, many environments** — build Docker image once, promote the same image through envs

---

## GitHub Actions — recommended structure

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  quality:
    name: Type Check + Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run typecheck          # tsc --noEmit
      - run: npm run lint               # eslint
      - run: npm run lint:deps          # dependency-cruiser (layer boundary check)

  test-unit:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: quality
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  test-integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: quality
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run db:migrate:test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db

  build:
    name: Docker Build
    runs-on: ubuntu-latest
    needs: [test-unit, test-integration]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Dockerfile (multi-stage, production-ready)

```dockerfile
# ---- Build stage ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build        # tsc → dist/

# ---- Production stage ----
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=builder /app/dist ./dist

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

---

## Environment promotion strategy

```
feature branch → PR → CI runs → merge to main
    main → build Docker image → tag with git SHA
        → deploy to staging (auto)
        → smoke tests pass → promote to production (manual approval)
```

In GitHub Actions, use **environments** with required reviewers for production:

```yaml
deploy-production:
  environment:
    name: production
    url: https://myapp.com
  needs: deploy-staging
```

---

## Secrets management

| Secret type | Where to store |
|-------------|---------------|
| Database URLs | GitHub Secrets → injected as env vars |
| API keys | GitHub Secrets or Vault |
| Docker registry creds | `GITHUB_TOKEN` (GHCR) or registry secret |
| Signing keys | GitHub Secrets — never in code |

**Rules:**
- Never commit `.env` files — add them to `.gitignore` and `.dockerignore`
- Use `.env.example` (with dummy values) to document required variables
- Rotate secrets after any accidental exposure

---

## Dependency boundary check (dependency-cruiser)

Add this to enforce that domain never imports from infrastructure:

```json
// .dependency-cruiser.json
{
  "forbidden": [
    {
      "name": "no-domain-imports-infrastructure",
      "severity": "error",
      "from": { "path": "src/.*/domain/" },
      "to": { "path": "src/.*/infrastructure/" }
    },
    {
      "name": "no-domain-imports-application",
      "severity": "error",
      "from": { "path": "src/.*/domain/" },
      "to": { "path": "src/.*/application/" }
    }
  ]
}
```

```json
// package.json
"scripts": {
  "lint:deps": "depcruise src --config .dependency-cruiser.json"
}
```

---

## Recommended npm scripts

```json
"scripts": {
  "build":           "tsc -p tsconfig.build.json",
  "typecheck":       "tsc --noEmit",
  "lint":            "eslint src --ext .ts",
  "lint:deps":       "depcruise src --config .dependency-cruiser.json",
  "test:unit":       "jest --testPathPattern=\\.spec\\.ts",
  "test:integration": "jest --testPathPattern=\\.int-spec\\.ts --runInBand",
  "test:e2e":        "jest --testPathPattern=\\.e2e-spec\\.ts --runInBand",
  "db:migrate":      "ts-node -r tsconfig-paths/register src/database/migrate.ts",
  "db:migrate:test": "cross-env NODE_ENV=test npm run db:migrate"
}
```

---

## GitLab CI variant

If using GitLab CI instead of GitHub Actions, the equivalent `.gitlab-ci.yml` structure follows the same jobs: `quality → test-unit → test-integration → build → deploy`. Ask for the GitLab variant explicitly if needed.
