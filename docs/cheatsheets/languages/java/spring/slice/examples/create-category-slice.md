---
title: Create Category Slice 
date: 2025-11-14
---



# Slice: CreateCategory – End-to-End

Goal: one **vertical slice** for creating a Category:

HTTP `POST /api/categories`  
→ app handler  
→ domain (`Category`, `Slug`, `CategoryId`)  
→ DB (JPA + Flyway)  
→ `ProblemDetail` errors.

Use this as a template for other create-* slices (CreateProduct, CreateTag, …).

---

## 1. Folder layout (feature: `category`)

```text
src/main/java/com/example/shop/
  web/
    category/
      CategoryController.java
      dto/
        request/
          CreateCategoryRequest.java
    support/
      errors/
        GlobalExceptionAdvice.java    # maps DuplicateCategory -> ProblemDetail

  app/
    category/
      CreateCategoryHandler.java      # use-case/service

  domain/
    category/
      Category.java                   # aggregate root
      CategoryId.java                 # typed ID
      Slug.java                       # value object (canonical key)
      CategoryRepository.java         # domain port
      exceptions/
        DuplicateCategoryException.java

  infra/
    jpa/
      category/
        CategoryEntity.java           # JPA entity
        SpringDataCategoryJpa.java    # extends JpaRepository<CategoryEntity, Long>
        JpaCategoryRepository.java    # implements domain.CategoryRepository
    mapping/
      category/
        CategoryMap.java              # MapStruct: domain <-> entity

src/main/resources/db/migration/
  V001__categories.sql
```

---

## 2. HTTP contract – CreateCategory

```text
POST /api/categories

Request JSON:
  {
    "name": "Books"
  }

Success:
  201 Created
  Location: /api/categories/{id}

Errors:
  400 Bad Request
    - invalid JSON
    - validation failure (name blank, too long, etc.)

  409 Conflict
    - type: /problems/duplicate-category
    - when slug derived from name already exists
```

---

## 3. Domain (business core)

### 3.1 `CategoryId` – typed ID

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

### 3.2 `Slug` – canonical key

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
  public static Slug fromName(String name) {
    String base = Objects.requireNonNull(name).trim().toLowerCase(Locale.ROOT);
    base = base.replaceAll("\\s+", "-").replaceAll("[^a-z0-9-]", "");
    base = base.replaceAll("(^-+|-+$)", "");
    return new Slug(base);
  }

  /** Wrap canonical persisted slug. */
  static Slug fromCanonical(String persisted) { return new Slug(persisted); }
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
    return new Category(null, n, Slug.fromName(n), Objects.requireNonNull(now));
  }

  /** Restore existing from persistence. */
  public static Category rehydrate(
      CategoryId id, String name, String slug, Instant createdAt
  ) {
    return new Category(
        Objects.requireNonNull(id),
        name,
        Slug.fromCanonical(slug),
        Objects.requireNonNull(createdAt)
    );
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

### 3.4 `CategoryRepository` port (only what this slice needs)

```java
// domain/category/CategoryRepository.java
package com.example.shop.domain.category;

public interface CategoryRepository {
  boolean existsBySlug(Slug slug);
  Category save(Category category);
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
package com.example.shop.infra.jpa.category;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.Instant;

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
package com.example.shop.infra.jpa.category;

import org.springframework.data.jpa.repository.JpaRepository;

public interface SpringDataCategoryJpa extends JpaRepository<CategoryEntity, Long> {
  boolean existsBySlug(String slug);
}
```

### 4.4 MapStruct mapper (domain ↔ entity)

```java
// infra/mapping/category/CategoryMap.java
package com.example.shop.infra.mapping.category;

import com.example.shop.domain.category.Category;
import com.example.shop.domain.category.CategoryId;
import com.example.shop.infra.jpa.category.CategoryEntity;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

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
}
```

### 4.5 Repo adapter (implements domain port)

```java
// infra/jpa/category/JpaCategoryRepositoryAdapter.java
package com.example.shop.infra.jpa.category;

import com.example.shop.domain.category.Category;
import com.example.shop.domain.category.CategoryRepository;
import com.example.shop.domain.category.Slug;
import com.example.shop.infra.mapping.category.CategoryMap;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class JpaCategoryRepositoryAdapter implements CategoryRepository {

  private final SpringDataCategoryJpa jpa;
  private final CategoryMap map;

  @Override
  public boolean existsBySlug(Slug slug) {
    return jpa.existsBySlug(slug.value());
  }

  @Override
  public Category save(Category category) {
    var entity = map.toEntityNew(category);
    if (entity.getCreatedAt() == null) {
      entity.setCreatedAt(category.createdAt());
    }
    var saved = jpa.save(entity);
    return map.toDomain(saved);
  }
}
```

---

## 5. App layer: use-case

```java
// app/category/CreateCategoryHandler.java
package com.example.shop.app.category;

import com.example.shop.domain.category.Category;
import com.example.shop.domain.category.CategoryId;
import com.example.shop.domain.category.CategoryRepository;
import com.example.shop.domain.category.Slug;
import com.example.shop.domain.category.exceptions.DuplicateCategoryException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.time.Instant;

@Service
@RequiredArgsConstructor
public class CreateCategoryHandler {

  private final CategoryRepository repo;

  public CategoryId handle(String name) {
    // derive slug to check duplicate
    Slug slug = Slug.fromName(name);

    if (repo.existsBySlug(slug)) {
      throw new DuplicateCategoryException(slug);
    }

    Category cat = Category.createNew(name, Instant.now());  // or TimeFacade later
    Category saved = repo.save(cat);
    return saved.id();
  }
}
```

---

## 6. Web: DTO + controller + error mapping

### 6.1 Request DTO

```java
// web/category/dto/request/CreateCategoryRequest.java
package com.example.shop.web.category.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record CreateCategoryRequest(
    @NotBlank
    @Size(max = 100)
    String name
) {}
```

### 6.2 Controller (CreateCategory only)

```java
// web/category/CategoryController.java
package com.example.shop.web.category;

import com.example.shop.app.category.CreateCategoryHandler;
import com.example.shop.domain.category.CategoryId;
import com.example.shop.web.category.dto.request.CreateCategoryRequest;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;

@RestController
@RequestMapping("/api/categories")
@RequiredArgsConstructor
public class CategoryController {

  private final CreateCategoryHandler createHandler;

  @PostMapping
  public ResponseEntity<Void> create(
      @Valid @RequestBody CreateCategoryRequest req,
      UriComponentsBuilder uris
  ) {
    CategoryId id = createHandler.handle(req.name());
    URI location = uris.path("/api/categories/{id}").build(id.value());
    return ResponseEntity.created(location).build();
  }
}
```

### 6.3 Global exception advice (`ProblemDetail`)

```java
// web/support/errors/GlobalExceptionAdvice.java
package com.example.shop.web.support.errors;

import com.example.shop.domain.category.exceptions.DuplicateCategoryException;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;
import java.util.Map;

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

## 7. Testing checklist (for this slice only)

* **Domain**

  * `Slug.fromName("  Sci Fi  ")` → `"sci-fi"`.
  * `Category.createNew(" Books ", now)` → name trimmed, slug `"books"`, createdAt = now.
* **App**

  * `CreateCategoryHandler.handle("Books")`:

    * when `existsBySlug` = true → throws `DuplicateCategoryException`.
    * when `existsBySlug` = false → calls `save`, returns non-null `CategoryId`.
* **Infra (@DataJpaTest)**

  * Saving a `CategoryEntity` with same slug twice → DB unique constraint violation (defensive).
* **Web (@WebMvcTest / @SpringBootTest)**

  * `POST /api/categories` with valid JSON → `201 Created` + `Location` header `/api/categories/{id}`.
  * `POST` with blank name → `400` ProblemDetail with validation info.
  * `POST` again with same name → `409` ProblemDetail `/problems/duplicate-category`.

---
