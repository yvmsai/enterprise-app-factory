---
name: testing-quality
description: Testing and quality-gate standards for Spring Boot + React apps - test pyramid, JUnit 5 + Mockito unit tests, @WebMvcTest slices, Testcontainers PostgreSQL integration tests, JaCoCo coverage, Vitest + Testing Library + MSW frontend tests, naming conventions and the gates code must pass before it counts as done. Use when writing tests, verifying a feature, or deciding whether work is complete.
---

# Testing and Quality Standards

## What to test, per feature

| Layer | Tool | Covers |
|---|---|---|
| Service unit | JUnit 5 + Mockito + AssertJ | Every business rule (BR-n in `docs/plan.md`), happy and failure paths |
| Controller slice | `@WebMvcTest` + `spring-security-test` | Status codes, validation errors (400 + `errors[]`), 401 without a token, 403 for the wrong role |
| Repository / integration | `@SpringBootTest` + Testcontainers PostgreSQL | Migrations apply, custom queries, constraints (unique, check), one end-to-end flow per feature |
| Frontend component | Vitest + Testing Library + MSW | Pages render all four states, forms validate, server errors show on fields |
| Smoke (optional) | Playwright | Log in → create → list, against Docker Compose |

## Backend conventions

- Test class `<ClassUnderTest>Test` for unit tests, `<Feature>IntegrationTest` for
  Testcontainers tests.
- Method names use `method_condition_expectedResult`, e.g.
  `create_duplicateSku_throwsConflict`.
- Arrange / Act / Assert blocks, with AssertJ (`assertThat`) assertions.
- Testcontainers via Spring Boot's service connection:

  ```java
  @SpringBootTest
  @Testcontainers
  class ProductIntegrationTest {
      @Container
      @ServiceConnection
      static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
  }
  ```

  Set `app.security.jwt-secret` in `src/test/resources/application-test.yml` with a
  dummy test-only value of at least 32 characters, and activate it with `@ActiveProfiles("test")`.
- Don't mock what you own at the integration level. Mock only external systems.
- JaCoCo (`jacoco-maven-plugin`) runs in `verify`, with a `check` rule of **≥ 80% line
  coverage** for `*Service` classes. Exclude DTOs, config and `Application`.

## Frontend conventions

- Test files sit next to the code: `ProductListPage.test.tsx`.
- Query by role and label (`getByRole('button', { name: /save/i })`), never by CSS class
  or test id unless there is no accessible alternative.
- Use `userEvent` (not `fireEvent`) for interaction.
- MSW handlers in `src/test/handlers.ts` mirror the API contract, including a 400
  problem-detail response.
- Wrap renders in a `renderWithProviders` helper (QueryClient with `retry: false`,
  MemoryRouter, AuthProvider).

## Quality gates: all must pass before a task is ticked

```bash
# backend (inside backend/)
./mvnw -B verify            # unit + integration tests + JaCoCo check

# frontend (inside frontend/)
npm run lint
npm test
npm run build
npm audit --omit=dev --audit-level=high
```

If local Java is older than 21, run the backend gate in a container:

```bash
docker run --rm -v "$PWD":/app -w /app -v /var/run/docker.sock:/var/run/docker.sock \
  -e TESTCONTAINERS_HOST_OVERRIDE=host.docker.internal \
  maven:3.9-eclipse-temurin-21 ./mvnw -B verify
```

## Reporting

At each checkpoint, report: test counts (passed/failed/skipped) for backend and frontend,
the coverage percentage, audit findings, and any test that was skipped or disabled, with
the reason. Never weaken or delete a failing test to make a gate pass.
