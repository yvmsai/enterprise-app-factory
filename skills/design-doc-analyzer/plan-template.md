# <App Name>: Implementation Plan

## Summary

<2–3 sentences>

## Roles

| Role | Can |
|---|---|
| ADMIN | ... |
| USER | ... |

## Entities

### <Entity>

| Field | Type | Required | Constraints |
|---|---|---|---|
| name | string(120) | yes | unique |

Relationships: <Entity> *—1 <Other> (many <Entity> belong to one <Other>)

## Business rules

- BR-1: ...
- BR-2: ...

## API endpoints

| Method | Path | Purpose | Roles |
|---|---|---|---|
| GET | /api/v1/<resources> | List, paginated | ADMIN, USER |

## Screens

| Route | Purpose | Roles |
|---|---|---|
| /login | Sign in | public |

## Non-functional requirements

- ...

## Assumptions

- A-1: ...

## Open questions

- Q-1: ... (BLOCKING / non-blocking)

## Out of scope

- ...

## Tasks

- [ ] T1 foundation: scaffolding, security, error handling, auth, login page
- [ ] T2 <feature>: migration, API, tests, UI, UI tests
- [ ] Tn packaging: Docker Compose, CI, render.yaml
