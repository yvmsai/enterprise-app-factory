# Enterprise App Factory

An AI software factory built as a [Claude Code](https://claude.com/claude-code) plugin.
Give it a **design document**, and it plans, builds, tests, containerizes and deploys a
full-stack enterprise application. Every layer follows its own **skill profile**, which
encodes enterprise engineering standards for that layer.

```
 design doc ──► PLAN ──► ARCHITECT ──► BUILD ──► VERIFY ──► RUN LOCALLY ──► DEPLOY FREE
                 │                        │          │            │               │
             you approve             backend      tests +     Docker          Render + Neon
                                     frontend     security    Compose         (you approve)
                                     database     gates
```

## Target stack

| Layer | Technology |
|---|---|
| Backend | Java 21, Spring Boot, Maven, Spring Security (JWT), Spring Data JPA, Flyway |
| Frontend | React, TypeScript, Vite, npm, TanStack Query, React Router, React Hook Form + Zod |
| Database | PostgreSQL 16 |
| Local run | Docker + Docker Compose (only Docker is required on your machine) |
| CI | GitHub Actions |
| Free hosting | Render (API as Docker web service + React as static site) and Neon (PostgreSQL) |

## Skill profiles

| Skill | Responsibility |
|---|---|
| [`design-doc-analyzer`](skills/design-doc-analyzer/SKILL.md) | Turns a design doc into requirements, entities, APIs, screens and an ordered task plan |
| [`solution-architecture`](skills/solution-architecture/SKILL.md) | Modular-monolith layout, repo structure, ADRs, cross-cutting decisions |
| [`api-contract`](skills/api-contract/SKILL.md) | REST conventions, error format, pagination, OpenAPI |
| [`postgres-data`](skills/postgres-data/SKILL.md) | Schema design, Flyway migrations, indexing, seed data |
| [`spring-boot-backend`](skills/spring-boot-backend/SKILL.md) | Package-by-feature layering, validation, error handling, logging, config |
| [`app-security`](skills/app-security/SKILL.md) | JWT auth, role-based access, CORS, secrets, OWASP checklist |
| [`react-frontend`](skills/react-frontend/SKILL.md) | Feature-based structure, data fetching, forms, accessibility |
| [`testing-quality`](skills/testing-quality/SKILL.md) | JUnit/Testcontainers, Vitest/Testing Library, coverage gates |
| [`docker-local`](skills/docker-local/SKILL.md) | Dockerfiles, Compose, local launch and health checks |
| [`ci-github-actions`](skills/ci-github-actions/SKILL.md) | Build/test/lint pipeline on every push and PR |
| [`free-cloud-deploy`](skills/free-cloud-deploy/SKILL.md) | Render Blueprint + Neon PostgreSQL deployment |

The [`app-factory-orchestrator`](agents/app-factory-orchestrator.md) agent runs the
pipeline and calls each skill in turn.

## Install

From GitHub (after you push this repo):

```
/plugin marketplace add <your-github-username>/enterprise-app-factory
/plugin install enterprise-app-factory@enterprise-app-factory
```

From a local checkout:

```bash
claude --plugin-dir ./enterprise-app-factory
```

## Usage

| Command | What it does |
|---|---|
| `/new-app <path-to-design-doc>` | Full pipeline: plan → build → test → run locally in Docker |
| `/add-feature <description or doc>` | Adds a feature to an app the factory generated |
| `/deploy-local` | (Re)builds and launches the app with Docker Compose |
| `/deploy-cloud` | Prepares and guides deployment to Render + Neon free tiers |

If another plugin uses the same command name, use the namespaced form, e.g.
`/enterprise-app-factory:new-app`.

Try it with the sample:

```
/new-app examples/inventory-lite-design.md
```

Write your own design doc from [`templates/design-doc-template.md`](templates/design-doc-template.md).

## Human checkpoints

The factory stops and asks you before it:

1. **Starts coding**: you approve the plan and architecture.
2. **Calls the build done**: you review the summary of code, tests and security checks.
3. **Deploys to the cloud**: you create the free accounts and enter secrets yourself.

The agent never handles your cloud passwords or database credentials. You enter them in
the hosting provider's dashboard.

## What a generated app looks like

```
<app-name>/
├── backend/                 Spring Boot (Maven) API
├── frontend/                React + Vite + TypeScript SPA
├── docs/
│   ├── design.md            copy of your design doc
│   ├── plan.md              task plan with progress checkboxes
│   └── adr/                 architecture decision records
├── docker-compose.yml
├── .env.example
├── render.yaml              Render Blueprint for free hosting
└── .github/workflows/ci.yml
```

## Free hosting notes

Free tiers change, so check the providers' current pricing pages. At the time of writing:

- **Render** free web services sleep after ~15 minutes idle, so the first request after a
  sleep takes 30–60 s. Static sites are free and always on.
- **Neon** free PostgreSQL scales to zero and doesn't expire. That's why it's used instead
  of Render's free PostgreSQL, which is deleted after 30 days.

## Roadmap

- [ ] More stacks (Node/NestJS, Angular)
- [ ] Self-healing loop: read deployed logs → propose fix → PR
- [ ] Playwright end-to-end smoke test after deploy
- [ ] More hosting targets (Koyeb, Fly.io, AWS)

## License

[MIT](LICENSE)
