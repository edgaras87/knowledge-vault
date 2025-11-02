---
title: Project Backbone
date: 2025-11-02
tags: 
    - spring
    - architecture
    - project-layout
    - best-practices
    - cheatsheet
summary: A compact, repeatable project layout for multi-entity Spring Boot applications using vertical slices.
aliases:
  - Project Backbone
---

# Project Layout Cheatsheet (edges ↔ core)

A compact, repeatable map for multi-entity Spring Boot apps. Use **vertical slices** (features) with the same rhythm in each slice: **web → app → domain**, plus **infra** and **resources**.

---

## Top-level

```
<app>/
├─ build.gradle | pom.xml
├─ README.md
├─ .editorconfig
├─ .gitignore
└─ src/
```

Keep the repo boring: one app per repo until you truly need multiple deployables.

---

## Package skeleton (repeat per slice)

```
src/main/java/com/edge/<app>/
├─ web/                # HTTP edge: controllers, DTOs, ProblemDetail
│  └─ {slice}/
│     ├─ {Slice}Controller.java
│     ├─ dto/
│     │  ├─ Create{Thing}Request.java
│     │  ├─ Update{Thing}Request.java
│     │  └─ {Thing}Response.java
│     └─ mapper/
│        └─ {Slice}WebMapper.java          # MapStruct, edge-owned mappings
├─ app/                # Use-cases (application layer)
│  └─ {slice}/
│     ├─ Create{Thing}UseCase.java
│     ├─ Update{Thing}UseCase.java
│     ├─ Query{Thing}UseCase.java
│     └─ {Slice}Service.java               # @Service, @Transactional, implements interfaces
├─ domain/             # Entities + repository ports (interfaces)
│  ├─ {slice}/
│  │  ├─ {Thing}.java
│  │  └─ {Thing}Repository.java
│  └─ common/
│     ├─ BaseEntity.java
│     ├─ Money.java / Quantity.java        # value objects (optional)
│     └─ DomainEvent.java (optional)
└─ infra/              # Adapters and wiring
   ├─ persistence/
   │  ├─ JpaConfig.java
   │  └─ SpecBuilders.java                  # Spring Data Specifications, projections
   ├─ mapping/
   │  └─ MapStructConfig.java               # global mapper rules
   └─ security/
      └─ SecurityConfig.java                # when you add auth
```

**Slices** are feature folders like `catalog/`, `inventory/`, `sales/`, `procurement/`. Add more as the domain grows.

---

## Resources & migrations

```
src/main/resources/
├─ application.yml
├─ application-<profile>.yml                # local, test, prod
├─ db/migration/
│  ├─ V1__init_<slice>.sql
│  ├─ V2__add_<constraint>.sql
│  └─ ...
└─ logback-spring.xml
```

**Truth lives in Flyway**. Entities follow the SQL, not vice-versa.

---

## Error handling (edge)

```
web/error/
├─ GlobalExceptionHandler.java              # @RestControllerAdvice → ProblemDetail
└─ ProblemTypes.java                        # stable type URIs (use properties for base URL)
```

* Translate exceptions **once** at the web edge.
* Use stable `type` URIs like `/problems/{slice}/{problem}`; base set via properties.

---

## Tests mirror `main`

```
src/test/java/com/edge/<app>/
├─ web/{slice}/{Slice}ControllerIT.java     # @SpringBootTest + MockMvc
├─ app/{slice}/{Slice}ServiceTest.java      # unit tests (Mockito) or @DataJpaTest if needed
└─ domain/{slice}/{Thing}RepositoryIT.java  # @DataJpaTest
src/test/resources/application-test.yml     # H2 or Testcontainers
```

---

## Responsibilities & allowed directions

* **web/**
  Speaks HTTP and DTOs. Handles validation, ProblemDetail, headers (`Location`, caching).
  **May depend on:** `app`, `domain` (types only), `infra.mapping` (MapStruct config).
  **Never:** return entities, call repositories directly.

* **app/**
  Orchestrates use-cases, transactions, cross-entity rules. Throws domain/app exceptions.
  **May depend on:** `domain` (entities, repositories).
  **Never:** know about HTTP, controllers, servlet stuff.

* **domain/**
  Entities, value objects, repository **ports** (Spring Data interfaces are fine).
  Self-contained invariants. No Spring web annotations.
  **Never:** know about `web` or `infra`.

* **infra/**
  Adapters (DB config, messaging, security), technical mappings/projections.
  **May depend on:** `domain` for entity classes.
  **Never:** call controllers or use DTOs.

Think **downward-only**: `web → app → domain` (and `infra` supports the stack).

---

## Naming rules (keep it predictable)

* DTOs: `{Action}{Thing}Request` (input), `{Thing}Response` (output).
  Scenario-first names: `AdjustStockRequest`, not `UpdateWarehouse`.
* Use-cases: verbs: `CreateProductUseCase`, `ListProductsUseCase`.
  Services implement multiple use-cases per slice: `CatalogService implements CreateProductUseCase, ...`
* Repositories: `{Thing}Repository`. No “DAO” suffixes.
* Controllers: `{Thing}Controller` or `{Slice}Controller` if it aggregates.

---

## Mappers (at the edge)

* Web mappers live under `web/{slice}/mapper`. They convert **DTO ↔ Command** and **Entity ↔ Response**.
* Global defaults in `infra/mapping/MapStructConfig.java`:

```java
@MapperConfig(
  componentModel = "spring",
  unmappedTargetPolicy = ReportingPolicy.ERROR,
  nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
```

Annotate each mapper with `@Mapper(config = MapStructConfig.class)`.

---

## Properties & configuration

* Put base URIs, feature flags, cache TTLs in `application.yml`.
* Bind via `@ConfigurationProperties(prefix = "app")` classes under `infra/` or `web/config/` (if HTTP-specific).
* Profiles: `local`, `test`, `prod`. Never commit secrets to YAML.

---

## Where future files go (decision table)

| Thing you’re adding                            | Folder                                  | Notes                                                   |
| ---------------------------------------------- | --------------------------------------- | ------------------------------------------------------- |
| New HTTP endpoint (CRUD, search)               | `web/{slice}/{Thing}Controller.java`    | DTOs + web mapper in same slice                         |
| Input/Output models for that endpoint          | `web/{slice}/dto/`                      | Validate with Bean Validation                           |
| Mapping between DTOs and entities              | `web/{slice}/mapper/`                   | Use MapStruct                                           |
| New business workflow                          | `app/{slice}/`                          | Add `*UseCase` + implement in service                   |
| Cross-entity rule (e.g., stock never negative) | `app/{slice}/{Slice}Service.java`       | Enforce in a @Transactional use-case                    |
| New entity                                     | `domain/{slice}/{Thing}.java`           | Keep invariants here                                    |
| Repository for entity                          | `domain/{slice}/{Thing}Repository.java` | Spring Data interface                                   |
| DB constraint / table                          | `resources/db/migration/V*__*.sql`      | One concern per migration                               |
| Global Jackson/CORS tweaks                     | `web/config/WebConfig.java`             | Keep edge-only concerns here                            |
| ProblemDetail mapping for new exception        | `web/error/GlobalExceptionHandler.java` | Add a method + constant in `ProblemTypes`               |
| Spring Security config                         | `infra/security/SecurityConfig.java`    | Keep HTTP authorizations in `web` annotations if needed |
| Projections/Specifications                     | `infra/persistence/`                    | Don’t leak to web/app                                   |
| Domain value object (Money, Quantity)          | `domain/common/`                        | Immutable, validated                                    |
| Scheduled job / batch                          | `app/{slice}/` or `infra/` (adapter)    | Use-case in `app`, triggers in `infra`                  |
| Integration client (SMTP, S3, Kafka)           | `infra/{integration}/`                  | One subpackage per integration                          |

---

## Adding a new slice (recipe)

1. Create `domain/{slice}` with entities + repositories.
2. Create `app/{slice}` with use-case interfaces + `{Slice}Service`.
3. Create `web/{slice}` with controller, DTOs, mapper.
4. Add Flyway migration(s).
5. Add tests mirroring `main`.

You can ship only web+app+domain for a slice; infra stays minimal until needed.

---

## When to split things further

* A service file passes ~400–500 lines or mixes unrelated workflows → split by verb or read/write (`OrderCommandService` vs `OrderQueryService`).
* DTO package gets crowded → subfolders per aggregate (`dto/product/*`, `dto/category/*`).
* Query logic turns gnarly → move complex queries to Specifications/Query classes in `infra/persistence`.

---

## Guardrails (the “don’t get weird” list)

* Don’t return entities from controllers. Always map to responses.
* Don’t inject repositories into controllers. Controllers talk to **use-cases** only.
* Don’t let `web` know about JPA annotations; don’t let `domain` know about HTTP.
* Don’t put shared junk in `domain/common/`. Only true, reusable domain primitives belong there.

---

## Tiny examples (paths)

* Create product endpoint
  `web/catalog/ProductController#create`
  DTOs in `web/catalog/dto/CreateProductRequest.java`
  Mapper in `web/catalog/mapper/ProductWebMapper.java`
  Use-case `app/catalog/CreateProductUseCase.java` → implemented by `CatalogService`.

* Adjust stock
  `web/inventory/StockItemController#adjust`
  `AdjustStockRequest` → `InventoryService.adjustStock(...)` → `StockRepository`.

