---
title: Vertical Slice Runbook 
date: 2025-11-11
---

# Vertical Slice Runbook (Spring Boot)

this “boilerplate” is worth writing—first for learning, later as reusable templates. Here’s a compact **Vertical Slice Runbook** you can keep as a cheatsheet and reuse slice after slice. It’s generic, but I’ll show filenames in your `web/app/domain/infra` style using **Category** as the example.

## 0) Define the slice (5–10 min)

Create a tiny note (copy this template every time).

```
# Slice: <Verb + Noun>  e.g., Create Category

## Story
As an <role>, I can <action> so that <outcome>.

## Contract
METHOD /api/<resource>
Request (JSON example):
{ ... }
Success:
<status> + headers + body example
Errors:
- <status> type:/problems/<slug>  (when …)
- <status> type:/problems/<slug>  (when …)

## Invariants
- rule 1
- rule 2

## Data (Flyway)
- up: tables/indexes/columns
- down (mental): how we’d revert

## Done
[ ] API works per contract
[ ] ProblemDetail for errors
[ ] Tests: domain / JPA / WebMvc / integration
[ ] One useful log/metric
```

---

## 1) Migration first (Flyway)

**Why:** own the data shape up front.

* File: `src/main/resources/db/migration/V001__init_categories.sql`

```sql
create table categories (
  id bigserial primary key,
  name varchar(100) not null,
  normalized_name varchar(100) not null,
  created_at timestamp not null default now(),
  unique (normalized_name)
);
```

> Common tweaks: add FK/indexes; never rely on JPA to create schema.

---

## 2) Domain rules (tiny and pure)

* File: `src/main/java/com/example/app/domain/category/Category.java`

```java
package com.example.app.domain.category;

import java.time.Instant;

public final class Category {
  private final String name;
  private final String normalizedName;
  private final Instant createdAt;

  public Category(String rawName) {
    var n = rawName == null ? "" : rawName.trim();
    if (n.isBlank()) throw new IllegalArgumentException("name.blank");
    this.name = n;
    this.normalizedName = n.toLowerCase();
    this.createdAt = Instant.now();
  }

  public String name() { return name; }
  public String normalizedName() { return normalizedName; }
  public Instant createdAt() { return createdAt; }
}
```

---

## 3) Persistence (JPA) + mapping

* Files:

  * `infra/jpa/category/CategoryEntity.java`
  * `infra/jpa/category/CategoryRepository.java`
  * `infra/mapping/category/CategoryMap.java`

```java
// infra/jpa/category/CategoryEntity.java
@Entity @Table(name="categories",
  indexes=@Index(name="uk_cat_norm", columnList="normalized_name", unique=true))
public class CategoryEntity {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
  @Column(nullable=false, length=100) String name;
  @Column(name="normalized_name", nullable=false, length=100) String normalizedName;
  @Column(name="created_at", nullable=false) Instant createdAt;
  // getters/setters
}

// infra/jpa/category/CategoryRepository.java
public interface CategoryRepository extends JpaRepository<CategoryEntity, Long> {
  boolean existsByNormalizedName(String normalizedName);
}

// infra/mapping/category/CategoryMap.java
@Mapper(componentModel = "spring")
public interface CategoryMap {
  @Mapping(target="id", ignore=true)
  @Mapping(target="createdAt", expression="java(java.time.Instant.now())")
  CategoryEntity toEntity(com.example.app.domain.category.Category domain);

  com.example.app.web.category.dto.response.CategoryResponse toResponse(CategoryEntity e);
}
```

---

## 4) Application/use-case (one handler per story)

* Files:

  * `app/category/CreateCategoryHandler.java`
  * `app/category/DuplicateCategoryException.java`

```java
@Service
public class CreateCategoryHandler {
  private final CategoryRepository repo;
  private final CategoryMap map;

  public CreateCategoryHandler(CategoryRepository repo, CategoryMap map) {
    this.repo = repo; this.map = map;
  }

  public Long handle(String name) {
    var normalized = name.trim().toLowerCase();
    if (repo.existsByNormalizedName(normalized)) throw new DuplicateCategoryException(normalized);
    var entity = map.toEntity(new Category(name));
    return repo.save(entity).getId();
  }
}

public class DuplicateCategoryException extends RuntimeException {
  private final String normalized; public DuplicateCategoryException(String n){ this.normalized=n; }
  public String normalized() { return normalized; }
}
```

---

## 5) Web edge (DTOs, controller, ProblemDetail)

* Files:

  * `web/category/dto/request/CreateCategoryRequest.java`
  * `web/category/dto/response/CategoryResponse.java`
  * `web/category/CategoryController.java`
  * `web/support/errors/GlobalExceptionAdvice.java`

```java
// DTOs
public record CreateCategoryRequest(@NotBlank String name) {}
public record CategoryResponse(Long id, String name) {}

// Controller
@RestController
@RequestMapping("/api/categories")
public class CategoryController {
  private final CreateCategoryHandler create;
  private final CategoryRepository repo;
  private final CategoryMap map;

  public CategoryController(CreateCategoryHandler create, CategoryRepository repo, CategoryMap map) {
    this.create = create; this.repo = repo; this.map = map;
  }

  @PostMapping
  public ResponseEntity<Void> create(@Valid @RequestBody CreateCategoryRequest req,
                                     UriComponentsBuilder uris) {
    var id = create.handle(req.name());
    return ResponseEntity.created(uris.path("/api/categories/{id}").build(id)).build();
  }

  @GetMapping
  public Page<CategoryResponse> list(@PageableDefault(size=20) Pageable p) {
    return repo.findAll(p).map(map::toResponse);
  }
}

// ProblemDetail
@RestControllerAdvice
public class GlobalExceptionAdvice {
  @ExceptionHandler(DuplicateCategoryException.class)
  public ProblemDetail duplicate(DuplicateCategoryException ex, HttpServletRequest req) {
    var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setTitle("Duplicate category");
    pd.setType(URI.create("/problems/duplicate-category"));
    pd.setDetail("Category already exists");
    pd.setProperty("normalized", ex.normalized());
    pd.setProperty("instance", req.getRequestURI());
    return pd;
  }
}
```

*(If you already have `JacksonConfig` for snake_case, it’ll apply automatically.)*

---

## 6) Tests (minimum set that keeps you brave)

* Files:

  * `domain/category/CategoryTest.java` (plain JUnit)
  * `infra/jpa/category/CategoryRepositoryTest.java` (`@DataJpaTest`)
  * `web/category/CategoryControllerTest.java` (`@WebMvcTest`)
  * `it/CategoryFlowIT.java` (SpringBootTest + Testcontainers)

**What to test**

* Domain: trim/blank/normalize rules.
* JPA: unique index behavior.
* WebMvc: `201 Created` + `Location`; `409` ProblemDetail on duplicate.
* Integration: POST then GET; ensure it really works through the stack.

---

## 7) Observability (one touch)

* Add one log in the use-case after save: `category.created` with `id` and `normalized`.
* Optional Micrometer counter: `categories_created_total`.

---

## 8) Post-slice refactor (time-boxed)

* 20–30 minutes to apply **max two** improvements you spotted (e.g., `ApiLinks`, tidy package names).
* Park all other ideas in `docs/structure-notes/research-debt.md` with a date.

---

# CRUD as slices (how to plan)

Treat each verb as **its own slice**:

* **Create Category** (done above)
* **List Categories** (pagination/filter)
* **Get Category by id**
* **Update Category name**
* **Delete Category** (decide rule: restrict if products exist)

Reads (get/list) can be bundled if trivial; writes deserve their own slice because they carry invariants and unique errors.

---

# Reusable templates (stash these in your vault)

## A) Contract template

```
METHOD /api/<resource>
Request:
{ ... }
Success:
<status> + headers + body example
Errors:
- <status> type:/problems/<slug> detail:"..."
Validation:
- field rules…
```

## B) Migration template

```sql
-- Vxxx__<short_name>.sql
-- up
create table <table>( ... );
create index ...;
alter table ...;

-- (keep notes in the PR for rollback plan)
```

## C) Controller skeleton

```java
@RestController
@RequestMapping("/api/<resource>")
class <Resource>Controller {
  // inject use-cases + queries
  @PostMapping
  ResponseEntity<Void> create(@Valid @RequestBody Create<Resource>Request req, UriComponentsBuilder uris) { ... }
  @GetMapping
  Page<<Resource>Response> list(@PageableDefault(size=20) Pageable p) { ... }
}
```

## D) ProblemDetail handler

```java
@ExceptionHandler(<DomainException>.class)
ProblemDetail handle(<DomainException> ex, HttpServletRequest req) {
  var pd = ProblemDetail.forStatus(<HttpStatus>);
  pd.setTitle("<title>");
  pd.setType(URI.create("/problems/<slug>"));
  pd.setDetail("<human message>");
  pd.setProperty("instance", req.getRequestURI());
  return pd;
}
```

## E) Test checklist

```
[ ] Domain: invariants
[ ] JPA: constraint/index behavior
[ ] WebMvc: happy path + error problem
[ ] Integration: end-to-end with Testcontainers
```

---

# When to add security in this runbook

After you finish **Create** and **List**, enable Spring Security and protect **mutations** (`POST/PUT/PATCH/DELETE`). Add one security-focused test per protected endpoint. Reads can remain public.

---

# That’s it

This gives you a start-to-finish recipe you can run today and reuse forever. Begin with **Create Category**, keep your contract note short, follow the steps, and cap refactors. Next slice, copy the same runbook and swap the nouns.
