---
name: spring-boot-backend
description: Enterprise standards for Java 21 + Spring Boot REST backends - package-by-feature structure, controller/service/repository layering, DTO records, Bean Validation, problem-detail error handling, transactions, logging, configuration and observability. Use when generating or modifying any server-side Java code.
---

# Spring Boot Backend Standards

Code patterns to copy are in [references/code-patterns.md](references/code-patterns.md).

## Package structure: package by feature

```
com.<org>.<app>
├── Application.java
├── common/
│   ├── config/          JpaAuditingConfig, OpenApiConfig, CorsConfig
│   ├── error/           GlobalExceptionHandler, NotFoundException, ConflictException, BusinessRuleException
│   ├── persistence/     BaseEntity
│   └── web/             PageResponse
├── security/            SecurityConfig, JwtService, AuthController, AppUser, AppUserRepository
└── <feature>/           e.g. product/
    ├── ProductController.java
    ├── ProductService.java
    ├── ProductRepository.java
    ├── Product.java                (entity)
    ├── ProductMapper.java
    └── dto/
        ├── CreateProductRequest.java
        ├── UpdateProductRequest.java
        └── ProductResponse.java
```

Features may call each other's **services**, never each other's repositories.

## Layer responsibilities

| Layer | Does | Never |
|---|---|---|
| Controller | HTTP mapping, `@Valid`, status codes, `@PreAuthorize` | Business logic, repository calls, returning entities |
| Service | Business rules, transactions, orchestration, logging of business events | HTTP types (`ResponseEntity`, `HttpServletRequest`) |
| Repository | Data access | Business logic |
| Mapper | Entity ↔ DTO conversion | I/O |

## Rules

- **Constructor injection only.** Fields are `private final`. No `@Autowired` on fields.
- **DTOs are Java records**, with Bean Validation annotations (`@NotBlank`, `@Size`,
  `@Positive`, `@Email`). Entities never leave the service layer.
- **Transactions:** annotate the service class `@Transactional(readOnly = true)` and each
  write method `@Transactional`.
- **Errors:** throw domain exceptions (`NotFoundException`, `ConflictException`,
  `BusinessRuleException`). A single `@RestControllerAdvice` maps them to
  `ProblemDetail`, following `api-contract`. No try/catch in controllers.
- **Optimistic locking:** catch `ObjectOptimisticLockingFailureException` in the handler
  and return 409.
- **Logging:** SLF4J with parameterized messages. INFO for business events (created,
  status changed), WARN for handled anomalies, ERROR only with the exception. Never log
  passwords, tokens or personal data.
- **Configuration:** `application.yml` holds shared defaults. `application-local.yml` and
  `application-prod.yml` hold the differences. Group custom settings under `app.*` and
  bind them with a `@ConfigurationProperties` record that has `@Validated`.
- **Server:** `server.port: ${PORT:8080}` (Render injects `PORT`), `server.shutdown: graceful`.
- **Observability:** Actuator exposes only `health` and `info`, with
  `management.endpoint.health.probes.enabled=true`.
- **Time:** use `Instant` for timestamps and `LocalDate` for dates. Inject a `Clock` bean
  where tests need to control time.
- **Lombok:** don't use it. Records and explicit constructors keep the code obvious.
- **Style:** methods under ~30 lines, no wildcard imports, `final` where it helps,
  meaningful names. No commented-out code or `System.out`.

## Definition of done for each feature

- [ ] Flyway migration applied, entity validates against it
- [ ] Controller, service, repository, mapper and DTOs follow this structure
- [ ] Every business rule from `docs/plan.md` enforced and covered by a test
- [ ] Endpoints visible and correct in Swagger UI
- [ ] `./mvnw verify` passes
