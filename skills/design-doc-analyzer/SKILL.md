---
name: design-doc-analyzer
description: Analyzes an application design document and produces a structured implementation plan (actors, roles, entities, relationships, API endpoints, screens, non-functional requirements, assumptions and an ordered task list) in docs/plan.md. Use at the start of building a new app or feature from a design or requirements document.
---

# Design Doc Analyzer

Turn a free-form design document into a plan precise enough that every later phase can
work from `docs/plan.md` without rereading the original doc.

## Process

1. Read the whole design document before writing anything.
2. Extract each section below. Quote the doc's own terms for entities and fields.
3. Wherever the doc is silent, decide using the defaults table and record the decision
   under **Assumptions**.
4. List real ambiguities under **Open questions**. Mark the ones that block work as
   `BLOCKING`.
5. Write `docs/plan.md` from [plan-template.md](plan-template.md).

## What to extract

| Section | Contents |
|---|---|
| Summary | 2–3 sentences: what the app does and for whom |
| Roles | Each role and what it may do (becomes Spring Security roles) |
| Entities | Name, fields (type, required, unique, constraints), relationships with cardinality |
| Business rules | Validations, state transitions, calculations, invariants (each one gets a test) |
| API endpoints | Method, path, purpose, roles allowed (detailed later by `api-contract`) |
| Screens | Route, purpose, data shown, actions, roles allowed |
| Non-functional | Performance, audit, data retention, browser support, accessibility |
| Tasks | Ordered vertical slices, one per feature, each with a checkbox |

## Defaults when the doc is silent

| Topic | Default |
|---|---|
| Authentication | Email + password login, JWT, roles `ADMIN` and `USER` |
| IDs | `BIGINT` identity |
| Audit fields | `created_at`, `updated_at` on every table, `version` for optimistic locking |
| Deletion | Hard delete, unless the doc mentions history or restore (then soft delete) |
| Lists | Paginated: page size 20, max 100, sortable by `createdAt` |
| Money | `NUMERIC(19,2)` in the DB, `BigDecimal` in Java, string in JSON |
| Dates | UTC `timestamptz` in the DB, ISO-8601 in JSON |

## Task ordering

1. `foundation`: scaffolding, security, error handling, base entity, auth endpoints, login page
2. One slice per entity or feature, ordering parents before children (Supplier before Product)
3. Cross-cutting screens: dashboard, reports
4. `packaging`: Docker, CI, deploy config

Each task gets this definition of done: migration, backend code, backend tests, frontend
code, frontend tests, all passing.

## Rules

- Never drop a requirement silently. If you postpone one, list it under **Out of scope**
  with the reason.
- Keep business rules testable: "Stock quantity can never go below 0" rather than "stock
  should be handled correctly".
