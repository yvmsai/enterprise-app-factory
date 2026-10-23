---
name: app-security
description: Application security standards for Spring Boot + React apps - stateless JWT authentication with Spring Security's resource server, BCrypt passwords, role-based authorization, CORS, secrets handling, security headers, dependency audits and an OWASP Top 10 review checklist. Use when implementing authentication or authorization, handling secrets or configuration, or reviewing code before release.
---

# Application Security Standards

Implementation patterns are in [references/security-config.md](references/security-config.md).

## Authentication

- Email + password login at `POST /api/v1/auth/login`, which returns
  `{ "accessToken": "...", "expiresIn": 1800 }`.
- Tokens are HS256 JWTs signed with `JWT_SECRET` (at least 256 bits, from the
  environment) and validated by Spring Security's OAuth2 resource server. Don't write a
  custom JWT filter.
- Claims: `sub` = user id, `email`, `roles` = `["ADMIN"]`, `iat`, `exp`. Access token TTL
  is 30 minutes.
- Passwords are hashed with `BCryptPasswordEncoder` (strength 10+). They are never logged
  or returned.
- Login failures return the same 401 message whether the email or the password is wrong.
- Self-registration is off unless the design doc asks for it. The first ADMIN comes from
  local seed data, or from a bootstrap env var in prod (`ADMIN_EMAIL` + `ADMIN_PASSWORD`,
  applied once at startup if no admin exists).

## Authorization

- Default deny: `anyRequest().authenticated()`. The only public paths are `/api/v1/auth/**`,
  `/actuator/health/**`, `/v3/api-docs/**` and `/swagger-ui/**`.
- Use `@EnableMethodSecurity` and `@PreAuthorize("hasRole('ADMIN')")` on controller
  methods, following the roles table in `docs/plan.md`.
- Check ownership in the service ("users may only edit their own X"). Never trust an id
  sent by the client.

## Frontend token handling

- Keep the access token in memory (a React context or store), not in `localStorage`.
  Logging in again after a page reload is acceptable for v1. Record this in an ADR.
- On a 401 response, clear the token and redirect to `/login`.
- Hiding UI elements by role is only for usability. The backend enforces every rule.

## CORS and headers

- Allowed origins come from `FRONTEND_ORIGIN` (comma-separated). Never `*` when
  credentials or auth headers are in use.
- Keep Spring Security's default headers (X-Content-Type-Options, X-Frame-Options, HSTS
  over HTTPS).
- CSRF protection is disabled only because the API is stateless and uses bearer tokens,
  with no cookies. If you ever switch to cookies, re-enable it.

## Secrets

- No secrets in Git: not in code, `application*.yml`, `docker-compose.yml` or tests.
- `.env` is gitignored. `.env.example` lists every variable with a placeholder.
- Cloud secrets are entered by the user in the provider dashboard, never by the agent.
- Before every commit: `git diff --cached | grep -iE "(password|secret|api[_-]?key|token)\s*[:=]"`
  must show placeholders only.

## Release checklist (OWASP Top 10)

- [ ] A01 Broken access control: every endpoint has an explicit role rule. Tests cover
      403 for the wrong role and 401 with no token.
- [ ] A02 Cryptographic failures: BCrypt for passwords, HTTPS in prod, secret length ≥ 256 bits
- [ ] A03 Injection: only Spring Data or parameterized queries. No string-concatenated SQL or JPQL.
- [ ] A04 Insecure design: business rules enforced server-side
- [ ] A05 Misconfiguration: Actuator limited to health/info, no stack traces in responses
- [ ] A06 Vulnerable components: `npm audit --omit=dev` shows no high/critical issues.
      Dependencies are recent versions.
- [ ] A07 Auth failures: generic login error, token expiry enforced
- [ ] A08 Integrity: CI builds from the lockfile (`npm ci`) and the Maven wrapper
- [ ] A09 Logging: auth failures logged at WARN without credentials
- [ ] A10 SSRF: no endpoint fetches user-supplied URLs (or they're allow-listed)
