---
title: Slice Starter Steps
date: 2025-11-11
---



# Spring slice starter: tiny online shop

A small shop; use it to master Spring craft. Later you can reskin the same architecture for fintech.

Here’s a tight, copy-paste-ready path so you can start coding today and ramp cleanly.

## 7-day starter plan (tiny → sturdy)

**Day 1–2: Category slice (end-to-end)**

* Flyway schema, domain invariants, repo, use-case, controller, ProblemDetail.
* Minimal tests: one domain unit, one `@DataJpaTest`, one `@WebMvcTest`, one happy-path integration.

**Day 3–4: Products + pagination/filtering**

* Product CRUD (create/list), filter by category, simple search.
* Add Jackson snake_case for JSON; keep Java camelCase.

**Day 5: Cart slice**

* Add/Remove line items; quantity rules; idempotent “add same item” merges quantity.

**Day 6: Web-edge hygiene**

* `ApiLinks`, `ProblemLinks`, `UriFactory`, global exception advice.
* Actuator health/info; structured logging baseline.

**Day 7: Security baseline**

* Basic/JWT auth, ADMIN vs USER; protect “create” endpoints.

---

## Folder shape (feature-oriented layered)

```
src/main/java/com/example/shop/
  web/
    category/...
    product/...
    cart/...
    support/{errors,links}/...
  app/
    category/...
    product/...
    cart/...
  domain/
    category/...
    product/...
    cart/...
    common/{Money, Quantity, ...}
  infra/
    jpa/...
    mapping/...
    config/{JacksonConfig, AppProps, SecurityConfig}
src/main/resources/db/migration/
```

---

## First slice: Category (minimal but solid)

## 1) Flyway

`src/main/resources/db/migration/V001__init.sql`

```sql
create table categories (
  id bigserial primary key,
  name varchar(100) not null,
  normalized_name varchar(100) not null,
  created_at timestamp not null default now(),
  unique (normalized_name)
);
```

## 2) Domain

`src/main/java/com/example/shop/domain/category/Category.java`

```java
package com.example.shop.domain.category;

import java.time.Instant;
import java.util.Objects;

public class Category {
  private Long id;
  private final String name;
  private final String normalizedName;
  private final Instant createdAt;

  public Category(String name) {
    String n = Objects.requireNonNull(name, "name").trim();
    if (n.isBlank()) throw new IllegalArgumentException("name.blank");
    this.name = n;
    this.normalizedName = n.toLowerCase();
    this.createdAt = Instant.now();
  }

  // factory for rehydration
  public static Category rehydrate(Long id, String name, String normalizedName, Instant createdAt) {
    Category c = new Category(name);
    c.id = id; // package-private setter if you prefer
    return new CategoryWith(id, name, normalizedName, createdAt);
  }

  // compact immutable holder
  private static final class CategoryWith extends Category {
    private final Long id;
    private final Instant createdAt;
    private CategoryWith(Long id, String name, String norm, Instant at) {
      super(name);
      super.id = id;
      super.normalizedName = norm;
      super.createdAt = at;
      this.id = id; this.createdAt = at;
    }
  }

  public Long id() { return id; }
  public String name() { return name; }
  public String normalizedName() { return normalizedName; }
  public Instant createdAt() { return createdAt; }
}
```

*(If you prefer JPA entities in `infra.jpa`, keep domain as POJOs/value objects and map in/out with MapStruct.)*

## 3) JPA entity + repo

`infra/jpa/category/CategoryEntity.java`

```java
@Entity @Table(name="categories", indexes=@Index(name="uk_categories_norm", columnList="normalized_name", unique=true))
public class CategoryEntity {
  @Id @GeneratedValue(strategy=IDENTITY) Long id;
  @Column(nullable=false, length=100) String name;
  @Column(name="normalized_name", nullable=false, length=100) String normalizedName;
  @Column(name="created_at", nullable=false) Instant createdAt;
  // getters/setters
}
```

`infra/jpa/category/CategoryRepository.java`

```java
public interface CategoryRepository extends JpaRepository<CategoryEntity, Long> {
  boolean existsByNormalizedName(String normalizedName);
}
```

## 4) MapStruct mapper

`infra/mapping/category/CategoryMap.java`

```java
@Mapper(componentModel = "spring")
public interface CategoryMap {
  @Mapping(target="id", ignore=true)
  @Mapping(target="createdAt", expression="java(java.time.Instant.now())")
  CategoryEntity toEntity(com.example.shop.domain.category.Category domain);

  com.example.shop.web.category.dto.response.CategoryResponse toResponse(CategoryEntity e);
}
```

## 5) App use-case

`app/category/CreateCategoryHandler.java`

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
    if (repo.existsByNormalizedName(normalized)) {
      throw new DuplicateCategoryException(normalized);
    }
    var entity = map.toEntity(new Category(name));
    return repo.save(entity).getId();
  }
}
```

`app/category/DuplicateCategoryException.java`

```java
public class DuplicateCategoryException extends RuntimeException {
  private final String normalized;
  public DuplicateCategoryException(String normalized) { this.normalized = normalized; }
  public String normalized() { return normalized; }
}
```

## 6) Web DTOs + controller

`web/category/dto/request/CreateCategoryRequest.java`

```java
public record CreateCategoryRequest(@NotBlank String name) {}
```

`web/category/dto/response/CategoryResponse.java`

```java
public record CategoryResponse(Long id, String name) {}
```

`web/category/CategoryController.java`

```java
@RestController @RequestMapping("/api/categories")
public class CategoryController {
  private final CreateCategoryHandler create;
  private final CategoryRepository repo;
  private final CategoryMap map;

  public CategoryController(CreateCategoryHandler create, CategoryRepository repo, CategoryMap map) {
    this.create = create; this.repo = repo; this.map = map;
  }

  @PostMapping
  public ResponseEntity<Void> create(@Valid @RequestBody CreateCategoryRequest req, UriComponentsBuilder uris) {
    Long id = create.handle(req.name());
    URI location = uris.path("/api/categories/{id}").build(id);
    return ResponseEntity.created(location).build();
  }

  @GetMapping
  public Page<CategoryResponse> list(@PageableDefault(size=20) Pageable pageable) {
    return repo.findAll(pageable).map(map::toResponse);
  }
}
```

## 7) ProblemDetail advice (global)

`web/support/errors/GlobalExceptionAdvice.java`

```java
@RestControllerAdvice
public class GlobalExceptionAdvice {

  @ExceptionHandler(DuplicateCategoryException.class)
  public ProblemDetail duplicate(DuplicateCategoryException ex, HttpServletRequest req) {
    var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setTitle("Duplicate category");
    pd.setDetail("Category with same name already exists.");
    pd.setType(URI.create("/problems/duplicate-category"));
    pd.setProperty("normalized", ex.normalized());
    pd.setProperty("instance", req.getRequestURI());
    return pd;
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ProblemDetail invalid(MethodArgumentNotValidException ex, HttpServletRequest req) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    pd.setType(URI.create("/problems/validation-error"));
    pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
      .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
      .toList());
    pd.setProperty("instance", req.getRequestURI());
    return pd;
  }
}
```

## 8) Jackson snake_case (optional now; add Day 3)

`infra/config/JacksonConfig.java`

```java
@Configuration
public class JacksonConfig {
  @Bean
  Jackson2ObjectMapperBuilderCustomizer jsonCase() {
    return b -> b.propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
  }
}
```

## 9) Minimal tests you’ll actually run

* **Domain:** blank/trim/normalize name.
* **Repo (`@DataJpaTest`):** unique constraint triggers on same normalized name.
* **Web (`@WebMvcTest`):** 201 Created with Location; 409 ProblemDetail on duplicate.
* **Integration:** Boot test with Testcontainers Postgres; hit POST+GET path once.

---

## Definition of Done (per slice)

* Works locally with Flyway.
* Returns correct HTTP codes and ProblemDetail for errors.
* Has 3–4 fast tests.
* Logs one useful event (`category.created`).
* No stringy URIs in controllers (once you add `ApiLinks`/`UriFactory`).

---

## When you’re ready to “go fintech”

Fork the repo and rename the nouns:

* Category → AccountType,
* Product → Instrument,
* Cart → TransactionBatch,
* Order → Transfer,
* Price/Discount → Fees/FX/Interest,
  keeping the same layering, migrations, ProblemDetail, idempotency, and tests.

You’re set. Build the Category slice now; don’t overthink it. Each slice is a rep at the gym—stack enough reps and you’ll be able to rebuild this without me, with only your cheatsheets for class names.
