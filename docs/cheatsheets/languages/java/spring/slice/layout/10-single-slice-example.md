---
title: Single-Slice Example 
date: 2025-11-02
tags: 
    - spring
    - architecture
    - project-layout
    - best-practices
    - cheatsheet
summary: A compact Spring Boot project layout example for a single vertical slice (feature).
aliases:
    - Single-slice example cheatsheet
---


# Spring Boot project layout: single slice example

```text
shopping-cart/
├─ build.gradle / pom.xml
├─ README.md
├─ .editorconfig
├─ .gitignore
└─ src
   ├─ main
   │  ├─ java/com/edge/shopping_cart
   │  │  ├─ web/                     # HTTP edge (controllers, request/response DTOs, exception translators)
   │  │  │  ├─ category/
   │  │  │  │  ├─ CategoryController.java
   │  │  │  │  ├─ dto/
   │  │  │  │  │  ├─ CreateCategoryRequest.java
   │  │  │  │  │  └─ CategoryResponse.java
   │  │  │  │  └─ mapper/            # MapStruct mappers AT THE EDGE
   │  │  │  │     └─ CategoryWebMapper.java
   │  │  │  ├─ error/
   │  │  │  │  ├─ GlobalExceptionHandler.java   # @RestControllerAdvice → ProblemDetail
   │  │  │  │  └─ ProblemTypes.java             # optional: catalog of problem "type" URIs / codes
   │  │  │  └─ config/
   │  │  │     └─ WebConfig.java                # CORS, Jackson tweaks, message converters if needed
   │  │  ├─ app/                      # Use-cases (application layer): orchestrates domain + repos
   │  │  │  ├─ category/
   │  │  │  │  ├─ CreateCategoryUseCase.java
   │  │  │  │  ├─ UpdateCategoryUseCase.java
   │  │  │  │  ├─ DeleteCategoryUseCase.java
   │  │  │  │  └─ CategoryService.java         # @Service, @Transactional, implements use-cases
   │  │  │  └─ exception/
   │  │  │     ├─ CategoryNotFoundException.java
   │  │  │     └─ DuplicateCategoryException.java
   │  │  ├─ domain/                  # Business objects + repository ports (interfaces)
   │  │  │  ├─ category/
   │  │  │  │  ├─ Category.java
   │  │  │  │  └─ CategoryRepository.java      # Spring Data interface (port)
   │  │  │  └─ common/
   │  │  │     └─ BaseEntity.java              # optional: id, createdAt, updatedAt
   │  │  └─ infra/                   # Adapters: DB config, security, integrations (SMTP, S3, etc.)
   │  │     ├─ persistence/
   │  │     │  └─ JpaConfig.java               # Naming strategies, batch size, etc. (if needed)
   │  │     ├─ security/
   │  │     │  └─ SecurityConfig.java          # Only if/when you add security
   │  │     └─ mapping/
   │  │        └─ MapStructConfig.java         # Global MapStruct defaults (optional)
   │  └─ resources
   │     ├─ application.yml
   │     ├─ application-local.yml              # local overrides
   │     ├─ db/migration/                      # Flyway: V1__init.sql, V2__add_unique_category.sql, ...
   │     └─ logback-spring.xml                 # sane logging (JSON or pattern)
   └─ test
      ├─ java/com/edge/shopping_cart
      │  ├─ web/category/
      │  │  └─ CategoryControllerIT.java       # @SpringBootTest + @AutoConfigureMockMvc
      │  ├─ app/category/
      │  │  └─ CategoryServiceTest.java        # @DataJpaTest or @ExtendWith(SpringExtension)
      │  └─ domain/category/
      │     └─ CategoryRepositoryIT.java       # @DataJpaTest
      └─ resources/
         └─ application-test.yml               # H2 or Testcontainers config
```

### Why this works (and where each thing lives)

* `web/` is the outermost HTTP shell:

  * Controllers, request/response **DTOs**, `@RestControllerAdvice`, and **edge mappers** (MapStruct).
  * Only talks in HTTP types and DTOs; never leaks JPA entities back to clients.
* `app/` holds the **use cases** and application rules:

  * `*UseCase` interfaces + `CategoryService` implementation.
  * Throws **domain/application exceptions**. Mark methods `@Transactional`.
* `domain/` is the boring, stable core:

  * Entities and **repository interfaces** (ports).
  * Keep it persistence-agnostic in spirit; Spring Data lives here fine because it’s an interface.
* `infra/` is wiring and adapters:

  * JPA config, security, external systems (email, S3, payment, Kafka).
  * When you add an external integration, create a subpackage per integration.
* `resources/db/migration/` is Flyway:

  * SQL scripts define schema truth. The entity should match the migration, not the other way around.
* `test/` mirrors `main/`:

  * Unit tests for `app/` services, slice tests for repos (`@DataJpaTest`), and integration tests for controllers.

### Category example (where your files go)

* `web/category/dto/CreateCategoryRequest.java`
* `web/category/dto/CategoryResponse.java`
* `web/category/mapper/CategoryWebMapper.java`
* `web/category/CategoryController.java`
* `app/category/CreateCategoryUseCase.java`, `UpdateCategoryUseCase.java`, `CategoryService.java`
* `app/exception/DuplicateCategoryException.java`, `CategoryNotFoundException.java`
* `domain/category/Category.java`, `CategoryRepository.java`
* `web/error/GlobalExceptionHandler.java` (translates exceptions → ProblemDetail)

### MapStruct: edge mappers live at the edge

Keep mappers with the layer that **owns the types**:

* Web mappers in `web/.../mapper` convert `Request ↔ Command` and `Entity ↔ Response`.
* If you ever add **persistence DTOs** (e.g., raw projections), add a mapper in `infra/mapping/` for that boundary.

Optional global config:

```java
// infra/mapping/MapStructConfig.java
@Configuration
@MapperConfig(
  componentModel = "spring",
  unmappedTargetPolicy = ReportingPolicy.ERROR,
  nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public class MapStructConfig {}
```

Then each mapper: `@Mapper(config = MapStructConfig.class)`.

### ProblemDetail handler location

* Put `GlobalExceptionHandler` in `web/error/`.
* It depends on HTTP concerns (`ProblemDetail`, `HttpStatus`), so it’s an edge concern.

### Configuration split

* `application.yml` → defaults.
* `application-local.yml` → dev overrides (DB URL, logging).
* `application-test.yml` → tests (H2 or Testcontainers).
* Use Spring profiles (`local`, `test`, `prod`).

### Flyway starter pack

`src/main/resources/db/migration/V1__init.sql`

```sql
CREATE TABLE category (
  id   BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL UNIQUE
);
```

`V2__trim_ci_unique_index.sql` (optional hardening)

```sql
-- Postgres example: case/trim-insensitive index
CREATE UNIQUE INDEX uq_category_name_norm ON category (lower(btrim(name)));
```

### Testing layout (pragmatic)

* Controller IT: `@SpringBootTest` + MockMvc, assert `Location` header and JSON body.
* Service tests: plain JUnit + Mockito **or** `@DataJpaTest` if you want real DB behavior.
* Repository tests: `@DataJpaTest` to lock in query behavior/constraints.

### Package names vs. folders

Java packages mirror folders 1:1. Keep them short and semantic:

* `com.edge.shopping_cart.web.category`
* `com.edge.shopping_cart.app.category`
* `com.edge.shopping_cart.domain.category`
* `com.edge.shopping_cart.infra.persistence`

### TL;DR rules of placement

* **HTTP stuff** (controllers, ProblemDetail, web mappers) → `web/`
* **Use cases + transactions + domain exceptions** → `app/`
* **Entities + repository interfaces** → `domain/`
* **DB/security/external adapters/config** → `infra/`
* **Schema** → `resources/db/migration/`
* **Tests** mirror **main**


