---
name: postgres-data
description: PostgreSQL schema and data-access standards - naming, column types, constraints, indexes, Flyway migrations, seed data and JPA mapping rules. Use when creating or changing database tables, migrations, entities or repositories.
---

# PostgreSQL Data Standards

## The schema is owned by Flyway

- Migrations live in `backend/src/main/resources/db/migration`, named
  `V<n>__<description>.sql` (e.g. `V3__create_product.sql`).
- Hibernate never changes the schema: `spring.jpa.hibernate.ddl-auto=validate`.
- **Never edit a migration that has been applied.** Add a new one instead.
- Local-only seed data (demo users, sample rows) goes in
  `src/main/resources/db/seed/V1000__seed_local.sql`. It's enabled only in the `local`
  profile via `spring.flyway.locations=classpath:db/migration,classpath:db/seed`.

## Naming

| Object | Convention | Example |
|---|---|---|
| Table | singular snake_case | `purchase_order` |
| Column | snake_case | `unit_price` |
| Primary key | `id` | |
| Foreign key column | `<referenced_table>_id` | `supplier_id` |
| Constraint | `pk_`, `fk_`, `uq_`, `ck_` prefix | `uq_product_sku` |
| Index | `ix_<table>_<columns>` | `ix_product_supplier_id` |

## Standard table shape

```sql
CREATE TABLE product (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    sku         VARCHAR(64)    NOT NULL,
    name        VARCHAR(200)   NOT NULL,
    unit_price  NUMERIC(19, 2) NOT NULL,
    quantity    INTEGER        NOT NULL DEFAULT 0,
    supplier_id BIGINT         NOT NULL,
    version     BIGINT         NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),
    CONSTRAINT pk_product PRIMARY KEY (id),
    CONSTRAINT uq_product_sku UNIQUE (sku),
    CONSTRAINT fk_product_supplier FOREIGN KEY (supplier_id) REFERENCES supplier (id),
    CONSTRAINT ck_product_quantity_non_negative CHECK (quantity >= 0)
);

CREATE INDEX ix_product_supplier_id ON product (supplier_id);
```

## Rules

- Enforce invariants in the database too (`NOT NULL`, `UNIQUE`, `CHECK`, FKs). Validation
  in Java isn't enough on its own.
- Index every foreign key, and every column used in a frequent `WHERE` or `ORDER BY`.
- Types: `TEXT` / `VARCHAR(n)` for strings, `NUMERIC(19,2)` for money, `TIMESTAMPTZ` for
  instants, `DATE` for calendar dates, `BOOLEAN` for flags, `VARCHAR(32)` + `CHECK` for enums.
- `user` is a reserved word, so name the accounts table `app_user`.

## JPA mapping rules

- Entities extend a `BaseEntity` with `id`, `@Version version`, `createdAt` and
  `updatedAt` (see `spring-boot-backend` references).
- `@Enumerated(EnumType.STRING)`, never `ORDINAL`.
- Associations are `LAZY` (`@ManyToOne(fetch = FetchType.LAZY)`). Load what a use case
  needs with `@EntityGraph` or a `JOIN FETCH` query, to avoid N+1 queries.
- Avoid bidirectional `@OneToMany` unless the aggregate really owns its children.
- `spring.jpa.open-in-view=false`.
- Repositories extend `JpaRepository<Entity, Long>`. Use derived queries for simple
  lookups, `@Query` (JPQL) for anything else, and native SQL only for reporting.
- Use projections or DTO queries for read-heavy lists and dashboards.
