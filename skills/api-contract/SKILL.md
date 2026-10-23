---
name: api-contract
description: REST API contract standards shared by the Spring Boot backend and React frontend - URL and verb conventions, status codes, RFC 9457 problem-detail errors, pagination shape, OpenAPI documentation and generated TypeScript types. Use when designing, implementing or consuming any HTTP endpoint.
---

# API Contract Standards

## URLs and verbs

- Base path `/api/v1`. Plural kebab-case nouns: `/api/v1/purchase-orders`.
- Nest only for true ownership, at most one level: `/api/v1/purchase-orders/{id}/lines`.
- Actions that aren't CRUD use a sub-resource verb: `POST /api/v1/purchase-orders/{id}/submit`.
- JSON fields are `camelCase`.

| Operation | Verb + path | Success |
|---|---|---|
| List | `GET /resources?page=0&size=20&sort=createdAt,desc` | 200 + page |
| Get | `GET /resources/{id}` | 200 |
| Create | `POST /resources` | 201 + `Location` header + body |
| Full update | `PUT /resources/{id}` | 200 |
| Partial update | `PATCH /resources/{id}` | 200 |
| Delete | `DELETE /resources/{id}` | 204 |

## Errors: RFC 9457 Problem Details

Every error response is `application/problem+json`:

```json
{
  "type": "about:blank",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more fields are invalid",
  "instance": "/api/v1/products",
  "errors": [{ "field": "sku", "message": "must not be blank" }]
}
```

| Status | When |
|---|---|
| 400 | Validation failure, malformed JSON (include `errors[]`) |
| 401 | Missing or invalid token |
| 403 | Authenticated but not permitted |
| 404 | Resource not found |
| 409 | Uniqueness conflict, optimistic-lock conflict, invalid state transition |
| 422 | Request is well-formed but breaks a business rule |
| 500 | Unexpected error. Generic message only: never stack traces or SQL |

## Pagination response

```json
{ "content": [], "page": 0, "size": 20, "totalElements": 0, "totalPages": 0 }
```

Don't serialize Spring's `Page` directly. Its JSON shape isn't stable, so map it to this
record.

## Field types

| Kind | JSON |
|---|---|
| ID | number |
| Money / decimals | string, e.g. `"19.99"` (no float rounding) |
| Timestamp | ISO-8601 UTC string, e.g. `"2026-01-15T10:30:00Z"` |
| Date | `"2026-01-15"` |
| Enum | UPPER_SNAKE string |

## Documentation and types

- The backend publishes OpenAPI with springdoc at `/v3/api-docs` and Swagger UI at
  `/swagger-ui.html`. Annotate controllers with `@Tag`, and operations with `@Operation`
  when the name isn't self-explanatory.
- The frontend generates types from the running API rather than hand-writing them:

  ```bash
  npx openapi-typescript http://localhost:8080/v3/api-docs -o src/shared/api/schema.d.ts
  ```

  Add this as `npm run gen:api` and rerun it whenever an endpoint changes.
