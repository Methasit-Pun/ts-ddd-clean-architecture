---
name: ts-ddd-security-review
description: Security review for TypeScript DDD code — OWASP-mapped checks for injection, auth/authz, data exposure, and misconfiguration. Trigger when the user says "security review", "check for vulnerabilities", "is this secure?", "review auth code", or when changes touch authentication, authorization, input handling, external APIs, file uploads, or user-controlled data.
---

# Security Review — TypeScript DDD

## Checklist

**Injection (OWASP A03)**
- All DB queries use parameterized queries or ORM methods — no string interpolation in SQL
- Flag every raw query: `query()`, `$queryRaw()`, raw `createQueryBuilder().where()`
- No `child_process.exec()` with user-controlled input — use `execFile()` with argument array

**Auth & Authorization (OWASP A01, A07)**
- JWT secrets from env vars, not hardcoded; `expiresIn` is set
- Passwords hashed with `bcrypt`/`argon2` (min cost 10) — never `md5`, `sha1`, `sha256` alone
- Every protected endpoint has an auth guard
- Authorization checks at **use case level**, not only at the controller
- Resource ownership verified: `userId` from JWT, not from request body
- Can a non-admin call an admin use case by changing a request param?

**Sensitive Data (OWASP A02)**
- Passwords/secrets/PII not logged — check `console.log`, logger calls, error serializers
- Response DTOs exclude `passwordHash`, `internalId`, `adminNotes`
- DB connection strings and API keys in env vars; `.env` in `.gitignore`

**Input Validation (OWASP A03, A04)**
- Request bodies validated at presentation layer (`class-validator`, `zod`, `joi`)
- Validation happens before reaching the application layer
- Max length constraints on text fields; file uploads: type-checked, size-limited, outside webroot
- UUID/ID format validated before hitting the repository

**Security Misconfiguration (OWASP A05)**
- CORS not `origin: '*'` in production
- Helmet headers applied: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`
- Stack traces not exposed in production responses
- Debug endpoints (`/metrics`, `/health`, `/debug`) are internal-only or auth-protected

**Cryptography & Dependencies**
- `crypto.randomBytes()` for token generation — not `Math.random()`
- Secrets compared with `timingSafeEqual()` — not `===` (prevents timing attacks)
- No `rejectUnauthorized: false` in HTTP clients
- `npm audit` run — no high/critical unresolved vulnerabilities
- Rate limiting on auth endpoints (login, register, password reset)

## Severity

- **[CRITICAL]** — Exploitable without auth (SQLi, RCE, auth bypass)
- **[HIGH]** — Requires auth, high impact (privilege escalation, data leak)
- **[MEDIUM]** — Defense-in-depth issue, misconfiguration, limited scope
- **[LOW]** — Hardening improvement, no direct exploitability

## DDD-specific

- Domain events must not carry PII — events may reach external buses
- Value Objects (`Email`, `PhoneNumber`) validate format on construction — natural security boundary
- Aggregate IDs should be UUIDs — sequential IDs allow enumeration attacks

## Output format

Numbered list, most critical first. Each finding: **[SEVERITY] Title** → location + exploit scenario + fix.
