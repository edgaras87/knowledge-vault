---
title: Category Slice 
date: 2025-11-13
---

# Category Slice – End-to-End Blueprint

Goal: one **thin but solid slice** for `Category`, from HTTP → app → domain → DB → errors, using:

- `web/app/domain/infra` layers
- typed IDs (`CategoryId`)
- slug (`Slug`) as canonical key
- ProblemDetail for errors

Use this as reference for future slices (Product, Cart, etc.).

---

## 1. Folder layout (feature-oriented)

Feature: **category**

```text
src/main/java/com/example/shop/
  web/
    category/
      CategoryController.java
      dto/
        request/
          CreateCategoryRequest.java
        response/
          CategoryResponse.java
    support/
      errors/
        GlobalExceptionAdvice.java    # handles DuplicateCategory, validation, etc.

  app/
    category/
      CreateCategoryHandler.java     # use-case/service
      ListCategoriesHandler.java     # optional

  domain/
    category/
      Category.java                  # aggregate root
      CategoryId.java                # typed ID
      Slug.java                      # value object (canonical key)
      CategoryRepository.java        # domain port
      exceptions/
        DuplicateCategoryException.java

  infra/
    jpa/
      category/
        CategoryEntity.java          # JPA entity
        SpringDataCategoryJpa.java   # extends JpaRepository<CategoryEntity, Long>
        JpaCategoryRepository.java   # implements domain.CategoryRepository

    mapping/
      category/
        CategoryMap.java             # MapStruct mapper: entity <-> domain, entity -> response

src/main/resources/db/migration/
  V001__categories.sql
```

---

## 2. HTTP contract (Category API)

Write this in your docs + use as tests base:

```text
POST /api/categories

Request:
  { "name": "Books" }

Success:
  201 Created
  Location: /api/categories/{id}        # or /api/categories/{slug}

Errors:
  400 Bad Request  – validation error
  409 Conflict     – type: /problems/duplicate-category

GET /api/categories

Query params:
  ?page=0&size=20&sort=name,asc

Response:
  200 OK
  [
    { "id": 1, "name": "Books" },
    { "id": 2, "name": "Electronics" }
  ]
```

---

## 3. Domain (business core)

### 3.1 `CategoryId` (typed ID)

```java
// domain/category/CategoryId.java
package com.example.shop.domain.category;

public record CategoryId(Long value) {
  public CategoryId {
    if (value == null || value <= 0) throw new IllegalArgumentException("id.invalid");
  }
  public static CategoryId of(long v) { return new CategoryId(v); }
}
```

### 3.2 `Slug` (canonical key)

```java
// domain/category/Slug.java
package com.example.shop.domain.category;

import java.util.Locale;
import java.util.Objects;
import java.util.regex.Pattern;

public record Slug(String value) {
  private static final Pattern ALLOWED = Pattern.compile("[a-z0-9-]+");

  public Slug {
    Objects.requireNonNull(value, "slug");
    if (!ALLOWED.matcher(value).matches()) throw new IllegalArgumentException("slug.invalid");
    if (value.length() > 100) throw new IllegalArgumentException("slug.tooLong");
  }

  /** Derive from user-facing name. */
  public static Slug of(String name) {
    String base = Objects.requireNonNull(name).trim().toLowerCase(Locale.ROOT);
    base = base.replaceAll("\\s+", "-").replaceAll("[^a-z0-9-]", "");
    base = base.replaceAll("(^-+|-+$)", "");
    return new Slug(base);
  }

  /** Wrap canonical persisted slug. */
  public static Slug ofSlug(String persisted) { return new Slug(persisted); }
}
```

### 3.3 `Category` aggregate

```java
// domain/category/Category.java
package com.example.shop.domain.category;

import java.time.Instant;
import java.util.Objects;

public final class Category {
  private final CategoryId id;     // null until persisted
  private final String name;
  private final Slug slug;
  private final Instant createdAt;

  private Category(CategoryId id, String name, Slug slug, Instant createdAt) {
    this.id = id;
    this.name = name;
    this.slug = slug;
    this.createdAt = createdAt;
  }

  /** Create new from user input. */
  public static Category createNew(String name, Instant now) {
    String n = requireNonBlank(name);
    return new Category(null, n, Slug.of(n), Objects.requireNonNull(now));
  }

  /** Restore existing from persistence. */
  public static Category rehydrate(
      CategoryId id, String name, String slug, Instant createdAt
  ) {
    return new Category(
        Objects.requireNonNull(id),
        name,
        Slug.ofSlug(slug),
        Objects.requireNonNull(createdAt)
    );
  }

  /** Rename – policy decision: re-derive slug or keep immutable. */
  public Category rename(String newName) {
    String n = requireNonBlank(newName);
    return new Category(this.id, n, Slug.of(n), this.createdAt);
  }

  private static String requireNonBlank(String s) {
    String t = Objects.requireNonNull(s).trim();
    if (t.isBlank()) throw new IllegalArgumentException("name.blank");
    return t;
  }

  public CategoryId id() { return id; }
  public String name() { return name; }
  public String slug() { return slug.value(); }
  public Instant createdAt() { return createdAt; }
}
```

### 3.4 `CategoryRepository` port

```java
// domain/category/CategoryRepository.java
package com.example.shop.domain.category;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import java.util.Optional;

public interface CategoryRepository {
  Category save(Category category);
  Optional<Category> findById(CategoryId id);
  Optional<Category> findBySlug(Slug slug);
  boolean existsBySlug(Slug slug);
  Page<Category> findAll(Pageable pageable);
}
```

### 3.5 Domain exception

```java
// domain/category/exceptions/DuplicateCategoryException.java
package com.example.shop.domain.category.exceptions;

import com.example.shop.domain.category.Slug;

public class DuplicateCategoryException extends RuntimeException {
  private final Slug slug;

  public DuplicateCategoryException(Slug slug) {
    super("Duplicate category: " + slug.value());
    this.slug = slug;
  }

  public Slug slug() { return slug; }
}
```

---

## 4. Infra: DB + JPA + mapping

### 4.1 Flyway migration

```sql
-- V001__categories.sql
create table categories (
  id         bigserial primary key,
  name       varchar(100) not null,
  slug       varchar(100) not null,
  created_at timestamp not null default now(),
  unique (slug)
);

create index ix_categories_created_at on categories(created_at);
```

### 4.2 JPA entity

```java
// infra/jpa/category/CategoryEntity.java
@Entity
@Table(name = "categories",
       indexes = @Index(name = "uk_categories_slug", columnList = "slug", unique = true))
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CategoryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 100)
  private String name;

  @Column(nullable = false, length = 100)
  private String slug;

  @Column(name = "created_at", nullable = false)
  private Instant createdAt;
}
```

### 4.3 Spring Data JPA interface

```java
// infra/jpa/category/SpringDataCategoryJpa.java
public interface SpringDataCategoryJpa extends JpaRepository<CategoryEntity, Long> {
  boolean existsBySlug(String slug);
  Optional<CategoryEntity> findBySlug(String slug);
}
```

### 4.4 MapStruct mapper

```java
// infra/mapping/category/CategoryMap.java
@Mapper(componentModel = "spring")
public interface CategoryMap {

  // Domain -> Entity for new saves
  @Mapping(target = "id", ignore = true)
  @Mapping(target = "slug", expression = "java(domain.slug())")
  CategoryEntity toEntityNew(Category domain);

  // Entity -> Domain (rehydrate)
  default Category toDomain(CategoryEntity e) {
    return Category.rehydrate(
        CategoryId.of(e.getId()),
        e.getName(),
        e.getSlug(),
        e.getCreatedAt()
    );
  }

  // Entity -> Web response DTO
  com.example.shop.web.category.dto.response.CategoryResponse toResponse(CategoryEntity e);
}
```

### 4.5 Repo adapter (implements domain port)

```java
// infra/jpa/category/JpaCategoryRepository.java
@Component
@RequiredArgsConstructor
public class JpaCategoryRepository implements CategoryRepository {

  private final SpringDataCategoryJpa jpa;
  private final CategoryMap map;

  @Override
  public Category save(Category category) {
    CategoryEntity entity = map.toEntityNew(category);
    if (entity.getCreatedAt() == null) {
      entity.setCreatedAt(category.createdAt());
    }
    CategoryEntity saved = jpa.save(entity);
    return map.toDomain(saved);
  }

  @Override
  public Optional<Category> findById(CategoryId id) {
    return jpa.findById(id.value()).map(map::toDomain);
  }

  @Override
  public Optional<Category> findBySlug(Slug slug) {
    return jpa.findBySlug(slug.value()).map(map::toDomain);
  }

  @Override
  public boolean existsBySlug(Slug slug) {
    return jpa.existsBySlug(slug.value());
  }

  @Override
  public Page<Category> findAll(Pageable pageable) {
    return jpa.findAll(pageable).map(map::toDomain);
  }
}
```

---

## 5. App layer: use-cases

### 5.1 Create handler

```java
// app/category/CreateCategoryHandler.java
@Service
@RequiredArgsConstructor
public class CreateCategoryHandler {

  private final CategoryRepository repo;

  public CategoryId handle(String name) {
    // derive slug to check duplicate
    Slug slug = Slug.of(name);
    if (repo.existsBySlug(slug)) {
      throw new DuplicateCategoryException(slug);
    }

    Category cat = Category.createNew(name, Instant.now());   // or via TimeFacade
    Category saved = repo.save(cat);
    return saved.id();
  }
}
```

### 5.2 List handler (optional)

```java
// app/category/ListCategoriesHandler.java
@Service
@RequiredArgsConstructor
public class ListCategoriesHandler {
  private final CategoryRepository repo;

  public Page<Category> handle(Pageable pageable) {
    return repo.findAll(pageable);
  }
}
```

---

## 6. Web: DTOs + controller + error mapping

### 6.1 DTOs

```java
// web/category/dto/request/CreateCategoryRequest.java
public record CreateCategoryRequest(
    @NotBlank String name
) {}

// web/category/dto/response/CategoryResponse.java
public record CategoryResponse(
    Long id,
    String name
) {}
```

### 6.2 Controller

```java
// web/category/CategoryController.java
@RestController
@RequestMapping("/api/categories")
@RequiredArgsConstructor
public class CategoryController {

  private final CreateCategoryHandler createHandler;
  private final ListCategoriesHandler listHandler;
  private final SpringDataCategoryJpa jpa;      // for simple listing via entity
  private final CategoryMap map;

  @PostMapping
  public ResponseEntity<Void> create(
      @Valid @RequestBody CreateCategoryRequest req,
      UriComponentsBuilder uris
  ) {
    CategoryId id = createHandler.handle(req.name());
    URI location = uris.path("/api/categories/{id}").build(id.value());
    return ResponseEntity.created(location).build();
  }

  @GetMapping
  public Page<CategoryResponse> list(@PageableDefault(size = 20) Pageable pageable) {
    // simple: use JPA + mapper to response
    return jpa.findAll(pageable).map(map::toResponse);
  }
}
```

### 6.3 Global exception advice (`ProblemDetail`)

```java
// web/support/errors/GlobalExceptionAdvice.java
@RestControllerAdvice
public class GlobalExceptionAdvice {

  @ExceptionHandler(DuplicateCategoryException.class)
  public ProblemDetail duplicate(DuplicateCategoryException ex, HttpServletRequest req) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setTitle("Duplicate category");
    pd.setType(URI.create("/problems/duplicate-category"));
    pd.setDetail("Category with the same slug already exists.");
    pd.setProperty("slug", ex.slug().value());
    pd.setProperty("instance", req.getRequestURI());
    return pd;
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ProblemDetail validation(MethodArgumentNotValidException ex, HttpServletRequest req) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    pd.setType(URI.create("/problems/validation-error"));
    pd.setProperty("instance", req.getRequestURI());
    pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
        .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
        .toList());
    return pd;
  }
}
```

---

## 7. Testing checklist for this slice

**Domain unit tests (`domain/category`)**

* `Category.createNew("  Sci Fi  ", now)` → slug `"sci-fi"`, name trimmed, createdAt set.
* `Category.rehydrate(id, name, slug, createdAt)` → uses `Slug.ofSlug`, no normalization.
* `Slug.of("  Sci Fi  ")` → `"sci-fi"`.

**App tests (`app/category`)**

* Create handler:

  * existing slug → `DuplicateCategoryException`
  * new name → repo.save called, returns `CategoryId`.

**Infra tests (`@DataJpaTest`)**

* Duplicate slug → DB uniqueness enforced.
* `SpringDataCategoryJpa.existsBySlug(...)` works.
* Mapping entity ↔ domain doesn’t lose data.

**Web tests (`@WebMvcTest`)**

* `POST /api/categories` valid body → `201 Created` + `Location` header.
* Duplicate → `409` ProblemDetail with `/problems/duplicate-category`.
* Invalid name → `400` ProblemDetail with validation info.

---

## 8. What this slice teaches (for future slices)

* **Domain owns**: `Category`, `CategoryId`, `Slug`, `CategoryRepository`, `DuplicateCategoryException`.
* **App owns**: orchestration/use-case logic (`CreateCategoryHandler`).
* **Infra owns**: persistence details (`CategoryEntity`, `SpringDataCategoryJpa`, `JpaCategoryRepository`).
* **Web owns**: HTTP contracts, DTO shapes, `ProblemDetail`.

Copy this pattern for the next feature (Product, Cart) and adjust:

* VOs (`Price`, `Quantity`, etc.)
* APIs (URLs & payloads)
* Repos (queries you need)

Each slice becomes one more repetition of the same muscle.
