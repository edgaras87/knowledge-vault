Absolutely.
Here’s a **complete, structured `index.md`** for your `docs/cheatsheets/domain/index.md`, written as a conceptual + practical cheatsheet.
It explains what the **domain layer** is, what belongs inside a *feature* package (like `category/`), and how to design it cleanly for a Spring Boot layered project.

---

# Domain Layer — Feature Cheatsheet

**Path:** `docs/cheatsheets/domain/index.md`
**Context:** Spring Boot layered project (`web/app/domain/infra`)
**Java 17+**, **MapStruct**, **JPA**, **Flyway**

---

## 0) Purpose in one line

> The **domain layer** holds your *business truths* — the rules, entities, and value objects that define what your system is *about*, independent of frameworks, databases, or HTTP.

---

## 1) Where it sits in your architecture

```
┌───────────────┐
│   Web Layer   │  ← controllers, DTOs, ProblemDetail, links
└──────┬────────┘
       │
┌──────▼────────┐
│   App Layer   │  ← use cases, handlers, orchestrates domain logic
└──────┬────────┘
       │
┌──────▼────────┐
│  Domain Layer │  ← business model (entities, VOs, policies)
└──────┬────────┘
       │
┌──────▼────────┐
│   Infra Layer │  ← persistence, external APIs, config
└───────────────┘
```

The domain knows nothing about Spring, HTTP, or databases — it’s **pure Java**.

---

## 2) Folder layout per feature

Example for `Category`:

```
src/main/java/com/example/shop/domain/
  category/
    Category.java              # aggregate root
    CategoryId.java            # typed ID
    Slug.java                  # value object
    CategoryRepository.java    # repository interface
  product/
    Product.java
    ProductId.java
    Money.java
    ProductRepository.java
  common/
    DomainException.java
    Quantity.java
    Money.java
```

Each *feature* (category, product, user, etc.) has its own small domain island.

---

## 3) What belongs in the domain layer

| Element                     | Description                                                                          | Example                                 |
| --------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------- |
| **Entity (Aggregate Root)** | Holds identity and business rules. Immutable except where rules allow.               | `Category`, `Product`, `User`           |
| **Value Object (VO)**       | Small, immutable type with value-based equality. Represents a concept, not identity. | `Slug`, `Money`, `Email`, `Quantity`    |
| **Typed ID**                | Value type for identity. Avoids mixing IDs across aggregates.                        | `CategoryId`, `ProductId`               |
| **Repository Interface**    | Abstraction for loading/persisting aggregates. Implementation lives in `infra/`.     | `CategoryRepository`                    |
| **Domain Service**          | Stateless business logic that doesn’t fit in one entity.                             | `DiscountPolicy`, `StockValidator`      |
| **Domain Event (optional)** | Record that something happened; publish internally.                                  | `CategoryCreated`, `ProductAddedToCart` |
| **Exception types**         | Express rule violations.                                                             | `DuplicateCategoryException`            |

---

## 4) What **doesn’t** belong here

* `@Entity`, `@Table`, `@Id` — those belong to `infra/jpa/`.
* Spring annotations (`@Service`, `@Component`, `@RestController`).
* JSON/Jackson annotations.
* HTTP status codes, DTOs, `ProblemDetail`.
* Database or file I/O code.

The domain must compile without a single Spring or JPA dependency.

---

## 5) Entity structure pattern

```java
public final class Category {
  private final CategoryId id;   // null until persisted
  private final String name;
  private final Slug slug;       // value object
  private final Instant createdAt;

  private Category(CategoryId id, String name, Slug slug, Instant createdAt) {
    this.id = id; this.name = name; this.slug = slug; this.createdAt = createdAt;
  }

  /** Create a brand-new category (runs rules). */
  public static Category createNew(String name, Instant now) {
    String n = requireNonBlank(name);
    return new Category(null, n, Slug.of(n), now);
  }

  /** Restore persisted category exactly as stored. */
  public static Category rehydrate(CategoryId id, String name, String slug, Instant createdAt) {
    return new Category(id, name, Slug.ofSlug(slug), createdAt);
  }

  public Category rename(String newName) {
    String n = requireNonBlank(newName);
    return new Category(this.id, n, Slug.of(n), this.createdAt);
  }

  private static String requireNonBlank(String s) {
    var t = Objects.requireNonNull(s).trim();
    if (t.isBlank()) throw new IllegalArgumentException("name.blank");
    return t;
  }

  public CategoryId id() { return id; }
  public String name() { return name; }
  public String slug() { return slug.value(); }
  public Instant createdAt() { return createdAt; }
}
```

---

## 6) Repository interface pattern

```java
public interface CategoryRepository {
  Category save(Category category);
  Optional<Category> findById(CategoryId id);
  Optional<Category> findBySlug(Slug slug);
  boolean existsBySlug(Slug slug);
  Page<Category> findAll(Pageable pageable);
}
```

Implementation lives in `infra/jpa/category/JpaCategoryRepository.java`.

---

## 7) Domain service example (optional)

```java
public class DiscountPolicy {
  public Money apply(Money price, int percent) {
    if (percent < 0 || percent > 100) throw new IllegalArgumentException("percent.invalid");
    return price.multiply(1 - percent / 100.0);
  }
}
```

Domain services have **no state** and **no frameworks**.

---

## 8) Domain value objects examples

| Concept      | Example                                              | Notes                        |
| ------------ | ---------------------------------------------------- | ---------------------------- |
| **Slug**     | `sci-fi-fantasy`                                     | URL-safe canonical key       |
| **Email**    | `record Email(String value)`                         | Lowercased, validated once   |
| **Money**    | `record Money(BigDecimal amount, Currency currency)` | Immutable, arithmetic inside |
| **Quantity** | `record Quantity(int value)`                         | Guards against negatives     |

VOs are *immutable*, *validated*, and compared by value, not identity.

---

## 9) Rehydration pattern (loading from DB)

> **Goal:** restore exact persisted state, without re-deriving derived fields.

Each aggregate provides a `rehydrate(...)` factory.
Infra adapters call it when mapping from JPA entities:

```java
Category.rehydrate(
  CategoryId.of(e.getId()),
  e.getName(),
  e.getSlug(),
  e.getCreatedAt()
);
```

Never run creation logic on load — that would re-normalize or alter canonical data.

---

## 10) Domain exceptions

Keep them expressive and domain-specific.

```java
public class DuplicateCategoryException extends RuntimeException {
  private final Slug slug;
  public DuplicateCategoryException(Slug slug) {
    super("Duplicate category: " + slug.value());
    this.slug = slug;
  }
  public Slug slug() { return slug; }
}
```

App layer catches and translates them into `ProblemDetail` later.

---

## 11) Testing strategy

| Level              | Purpose                            | Example                                 |
| ------------------ | ---------------------------------- | --------------------------------------- |
| **Unit**           | Test entity rules & value objects. | `createNew("Sci Fi")` → slug `"sci-fi"` |
| **Domain service** | Test policy logic in isolation.    | discount math, validation rules         |
| **App/service**    | Test orchestration.                | handler + repo mock                     |
| **Integration**    | Test full flow (with DB).          | create + read category                  |

Domain tests require **no Spring context**. Pure JUnit.

---

## 12) Dependency rules (clean boundaries)

* **Domain** depends on nothing (no imports from `app`, `infra`, or `web`).
* **App** depends on `domain` (calls its entities & repos).
* **Infra** depends on `domain` (implements its repos).
* **Web** depends on `app` (maps requests to use cases).

Violating this order = tangled architecture.

---

## 13) Common pitfalls

* Putting JPA annotations inside `domain/` (leaks persistence concerns).
* Using Lombok’s `@Data` on entities (bypasses invariants).
* Mixing creation logic and rehydration logic.
* Treating domain as DTOs (no behavior, only data).
* Forgetting typed IDs or value objects, leading to unsafe code.
* Recomputing derived fields (like slug) on every load.

---

## 14) Keep domain pure

Checklist before you commit:

✅ No `springframework.*` imports
✅ No `javax.persistence.*`
✅ No logging, HTTP, or serialization code
✅ Only Java + your own domain types

---

## 15) TL;DR summary card

> The **domain layer** is the heart of your app:
> *entities* that enforce rules, *value objects* that keep data honest, and *repositories* that abstract persistence.
>
> It should read like **business logic**, not plumbing code.
> When in doubt: if it’s about *what the system does*, it’s domain;
> if it’s about *how it’s done*, it’s elsewhere.

---

This `index.md` forms the **entry point** for all other domain sub-cheatsheets —
(`slug/`, `id/`, `money/`, `quantity/`, `exceptions/`, etc.).
Each feature can plug into this structure cleanly and grow independently.
