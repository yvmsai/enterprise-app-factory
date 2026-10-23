---
name: solution-architecture
description: Defines the architecture and repository skeleton for a Spring Boot + React + PostgreSQL enterprise app - modular monolith, package-by-feature, monorepo layout, environment profiles - and records decisions as ADRs. Use after planning and before writing application code, or when a structural decision must be made.
---

# Solution Architecture

## Default architecture: modular monolith

One Spring Boot API, one React SPA, one PostgreSQL database. Split into microservices only
when the design doc names a concrete need: independent scaling, separate teams, or
different release cycles. Record that decision in an ADR.

```
Browser ──► React SPA (static) ──HTTPS/JSON──► Spring Boot API ──JDBC──► PostgreSQL
                                   Bearer JWT
```

## Repository layout (monorepo)

```
<app-name>/
├── backend/                      Spring Boot, Maven wrapper
├── frontend/                     React + Vite + TypeScript, npm
├── docs/
│   ├── design.md
│   ├── plan.md
│   └── adr/0001-<decision>.md
├── docker-compose.yml
├── .env.example
├── render.yaml
├── .github/workflows/ci.yml
├── .gitignore
└── README.md
```

## Scaffolding

Generate rather than hand-write so dependency versions are current:

```bash
curl -sSf https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d javaVersion=21 \
  -d groupId=com.<org> -d artifactId=<app>-api -d name=<app>-api \
  -d packageName=com.<org>.<app> \
  -d dependencies=web,data-jpa,postgresql,flyway,validation,security,oauth2-resource-server,actuator,testcontainers \
  -o backend.zip && unzip -q backend.zip -d backend && rm backend.zip

npm create vite@latest frontend -- --template react-ts
```

Then add `springdoc-openapi-starter-webmvc-ui` to `pom.xml` (latest release from Maven
Central).

## Environments and profiles

| Environment | Spring profile | Config source |
|---|---|---|
| Developer machine / Docker Compose | `local` | `.env` file (gitignored) |
| CI | `test` | Testcontainers, no external config |
| Cloud | `prod` | Hosting provider's environment variables |

All environment-specific values come from environment variables. Nothing
environment-specific is hard-coded.

## Cross-cutting decisions (record each as an ADR)

1. `0001-modular-monolith.md`: architecture style
2. `0002-jwt-authentication.md`: stateless auth, see `app-security`
3. `0003-flyway-migrations.md`: schema owned by migrations, `ddl-auto=validate`
4. `0004-free-hosting.md`: Render + Neon, and the free-tier trade-offs

## ADR template

```markdown
# <number>. <Title>

Date: YYYY-MM-DD
Status: Accepted

## Context
<What forces this decision?>

## Decision
<What we chose.>

## Consequences
<Trade-offs, follow-ups, what becomes easier or harder.>
```
