---
name: ts-ddd-security-review
description: Security review for TypeScript DDD clean architecture code — checks for injection, auth/authz issues, data exposure, and insecure dependencies. Trigger when the user says "security review", "check for vulnerabilities", "is this secure?", "review auth code", "check this for OWASP issues", "audit this endpoint", or when changes touch authentication, authorization, input handling, external API calls, file uploads, or any user-controlled data. Also trigger before deploying to production or before a security audit.
---

# Security Review — TypeScript DDD

Systematic security check scoped to the concerns most relevant to TypeScript backend services built on clean architecture. Each finding maps to an OWASP category where applicable.

## Review by concern

### 1. Injection (OWASP A03)

**SQL / NoSQL injection**
- [ ] All queries use parameterized queries or ORM methods — no string interpolation in SQL
- [ ] Raw query methods (`query()`, `$queryRaw()`, `createQueryBuilder().where(raw)`) — flag each one

**Command injection**
- [ ] No `child_process.exec()` or `execSync()` with user-controlled input
- [ ] If shell commands are needed, use `execFile()` with an argument array, never template strings

---

### 2. Authentication and Authorization (OWASP A01, A07)

**Authentication**
- [ ] JWT secrets are loaded from environment variables, not hardcoded
- [ ] JWT `expiresIn` is set — tokens do not live forever
- [ ] Passwords are hashed with `bcrypt` or `argon2` (min cost factor 10) — never `md5`, `sha1`, `sha256` alone

**Authorization**
- [ ] Every protected endpoint has an auth guard
- [ ] Authorization checks happen at the **use case level**, not only at the controller
- [ ] Resource ownership is verified — `userId` from JWT, not from request body
- [ ] Privilege escalation check: can a non-admin user call an admin use case by changing a request param?

---

### 3. Sensitive Data Exposure (OWASP A02)

- [ ] Passwords, secrets, PII are never logged — check `console.log`, `logger.info`, error serializers
- [ ] Response DTOs do not include `passwordHash`, `internalId`, or `adminNotes`
- [ ] Database connection strings and API keys are in env vars, not committed to git
- [ ] `.env` is in `.gitignore`

---

### 4. Input Validation (OWASP A03, A04)

- [ ] Request bodies are validated at the presentation layer (e.g. `class-validator`, `zod`, `joi`)
- [ ] Validation happens **before** reaching the application layer
- [ ] Max length constraints on text fields — no unbounded string inputs to DB
- [ ] File uploads: type checked, size limited, stored outside webroot
- [ ] UUID/ID format validated before hitting the repository

---

### 5. Security Misconfiguration (OWASP A05)

- [ ] CORS is configured restrictively — `origin: '*'` in production is a red flag
- [ ] Helmet (or equivalent headers) applied: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`
- [ ] Stack traces and detailed errors are not exposed in production responses
- [ ] Debug endpoints (`/metrics`, `/health`, `/debug`) are either internal-only or auth-protected

---

### 6. Cryptography (OWASP A02)

- [ ] `crypto.randomBytes()` used for token generation — not `Math.random()`
- [ ] Secrets comparison uses `timingSafeEqual()` — not `===` (prevents timing attacks)
- [ ] No `rejectUnauthorized: false` in HTTP clients

---

### 7. Dependency vulnerabilities (OWASP A06)

- [ ] `npm audit` has been run — no high/critical unresolved vulnerabilities
- [ ] Dependencies are pinned or use lock files committed

---

### 8. Rate limiting and DoS (OWASP A04)

- [ ] Rate limiting applied to auth endpoints (login, register, password reset)
- [ ] File upload endpoints have size limits
- [ ] Expensive queries have pagination

---

## Severity classification

- **[CRITICAL]** — Exploitable without authentication (SQLi, RCE, auth bypass)
- **[HIGH]** — Requires authentication but has high impact (privilege escalation, data leak)
- **[MEDIUM]** — Defense-in-depth issue, misconfiguration, or limited-scope exposure
- **[LOW]** — Informational, hardening improvement, no direct exploitability

---

## Report format

```
## Security Review

### [CRITICAL] SQL injection via raw query
`src/orders/infrastructure/OrderRepository.ts` line 47 uses a raw query with
string interpolation. An attacker controlling the input can execute arbitrary SQL.
Fix: Use the ORM parameterized method or a prepared statement with placeholders.

### [HIGH] Authorization checked only at controller, not use case
CancelOrderUseCase does not verify that the requesting user owns the order.
An authenticated user can cancel any order by supplying a different orderId.
Fix: Pass requestingUserId into the use case and verify order.customerId.equals(requestingUserId).
```

---

## DDD-specific security notes

- **Domain events must not leak sensitive data** — events may be published to external buses; strip PII or encrypt sensitive fields before publishing
- **Value Objects validate their own format** — `Email`, `PhoneNumber`, `Url` VOs should throw on invalid input, providing a natural validation boundary
- **Aggregate IDs should be UUIDs** — sequential IDs allow enumeration attacks
