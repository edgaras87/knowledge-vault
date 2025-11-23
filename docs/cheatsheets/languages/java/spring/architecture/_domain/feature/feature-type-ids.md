---
title: Feature Typed IDs
date: 2025-11-12
tags: 
  - spring
  - spring-boot
  - architecture
  - domain
  - cheatsheet 
  
summary: Define and use typed IDs (value objects) per aggregate root in your Spring layered application to enhance type safety, clarity, and future-proofing of your domain model.
aliases:
  - Spring Domain Layer - Feature Typed IDs Cheatsheet
---


# Typed IDs — Practical Cheatsheet

- **Path:** `docs/cheatsheets/domain/id/index.md`
- **Context:** Spring layered app (`domain/app/infra/web`), Java 17+, JPA/Hibernate, MapStruct

---

## What & Why (in one screen)

* **Problem:** raw `Long`/`UUID` everywhere → easy to pass `User` ID into `Category` by mistake; painful to change key type later.
* **Solution:** wrap the key in a tiny **value type** per aggregate (e.g., `CategoryId`, `UserId`).
* **Benefits:**

  * **Compile-time safety** (no cross-type mixups)
  * **Self-documenting APIs** (`findById(CategoryId id)`)
  * **Future-proofing** (Long → UUID/ULID change is localized)
  * **Centralized behavior** (parsing, validation, toString)

---

## Where it lives

```
src/main/java/com/example/shop/domain/category/
  CategoryId.java      ← here (domain owns its ID)
  Category.java
```

Keep IDs **with their aggregates**. Only promote to `domain/common/` if truly generic.

---

## Minimal patterns (pick one)

### A) Long-backed ID (record)

```java
package com.example.shop.domain.category;

public record CategoryId(Long value) {
  public CategoryId {
    if (value == null || value <= 0) throw new IllegalArgumentException("id.invalid");
  }
  public static CategoryId of(long v) { return new CategoryId(v); }
  @Override public String toString() { return String.valueOf(value); }
}
```

### B) UUID-backed ID (record)

```java
package com.example.shop.domain.category;

import java.util.UUID;

public record CategoryId(UUID value) {
  public CategoryId {
    if (value == null) throw new IllegalArgumentException("id.null");
  }
  public static CategoryId newId() { return new CategoryId(UUID.randomUUID()); }
  public static CategoryId parse(String s) { return new CategoryId(UUID.fromString(s)); }
  @Override public String toString() { return value.toString(); }
}
```

*(ULID works similarly—good for sortable, URL-friendly strings.)*

---

## Domain usage

```java
public final class Category {
  private final CategoryId id;   // null (or Optional) until persisted
  // ...
  public CategoryId id() { return id; }
}
```

Repositories speak **typed IDs**:

```java
public interface CategoryRepository {
  Category save(Category c);
  boolean existsById(CategoryId id);
  Optional<Category> findById(CategoryId id);
}
```

---

## JPA mapping (infra)

### Long key schema

```sql
create table categories (
  id bigserial primary key,
  -- ...
);
```

### UUID key schema (Postgres)

```sql
create extension if not exists "uuid-ossp";
create table categories (
  id uuid primary key default uuid_generate_v4(),
  -- ...
);
```

### AttributeConverter (Long)

```java
@Converter(autoApply = false)
public class CategoryIdConverter implements AttributeConverter<CategoryId, Long> {
  public Long convertToDatabaseColumn(CategoryId id) { return id == null ? null : id.value(); }
  public CategoryId convertToEntityAttribute(Long v) { return v == null ? null : new CategoryId(v); }
}
```

### AttributeConverter (UUID)

```java
@Converter(autoApply = false)
public class CategoryIdUuidConverter implements AttributeConverter<CategoryId, UUID> {
  public UUID convertToDatabaseColumn(CategoryId id) { return id == null ? null : id.value(); }
  public CategoryId convertToEntityAttribute(UUID v) { return v == null ? null : new CategoryId(v); }
}
```

### JPA entity usage

```java
@Entity
@Table(name = "categories")
@Getter @Setter @NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CategoryEntity {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)         // for Long
  // @GeneratedValue                                             // for DB-generated UUID
  private Long id;                                            // or UUID id;

  // fields...
}
```

> Tip: Converters are handy if your **domain** stores `CategoryId` but your **entity** uses `Long/UUID`.
> In many projects you simply keep `Long/UUID` in the entity and wrap/unwrap at the mapper.

---

## MapStruct mapping

```java
@Mapper(componentModel = "spring")
public interface CategoryMap {

  // Entity -> Domain (rehydrate)
  default CategoryId toCategoryId(Long id) {
    return id == null ? null : new CategoryId(id);
  }
  default Long fromCategoryId(CategoryId id) {
    return id == null ? null : id.value();
  }

  // ...use helpers in your mapping methods
}
```

*(Do the UUID variant if you use UUID.)*

---

## JSON & API shape

* **Server-internal:** use `CategoryId` everywhere.
* **Over the wire:** return plain scalar (number/string). Don’t leak `value` wrappers to clients unless you want them to post typed IDs.

Example DTO:

```java
public record CategoryResponse(Long id, String name) { }        // or String for UUID
```

---

## Equality & collections

* For IDs, rely on the record’s value-based `equals/hashCode`.
* For **entities** (JPA), be careful with equals/hashCode (proxies).
  Use defaults or explicitly include only the ID once it’s assigned.

---

## Decision flow — when to introduce typed IDs

1. Will more than one aggregate have an ID? → **Yes** → Use typed IDs.
2. Could I accidentally pass the wrong ID type? → **Yes** → Use typed IDs.
3. Might key type change (Long → UUID/ULID)? → **Yes** → Use typed IDs.
4. Single tiny CRUD, throwaway? → **Maybe skip** (you can add later).

---

## Migration notes (Long → UUID later)

* **If you used typed IDs:** update only the ID record + mappers + schema. Call sites remain `CategoryId`.
* **If you used raw Long:** widespread signature changes. Pain.

Plan:

1. Add new UUID column, backfill (`UPDATE categories SET uuid = gen_random_uuid()`), dual-write, then swap primary key in a maintenance window.
2. Keep a mapping table for external references during transition if needed.

---

## Testing checklist

* `new CategoryId(0)` throws.
* `CategoryRepository.findById(CategoryId.of(123))` loads correctly.
* MapStruct mapping unwraps/wraps IDs as expected.
* If UUID: parsing invalid strings throws, `newId()` creates valid UUIDs.

---

## Pitfalls to avoid

* Serializing the whole `CategoryId` object in JSON unintentionally (keep DTO scalar).
* Over-engineering a **generic** ID type too early; start per-aggregate, extract common later if truly shared.
* Mixing typed IDs and raw primitives in the same API surface (be consistent).
* Forgetting `@Converter` or mapper helpers, causing NullPointerExceptions at the DB edge.

---

## TL;DR card

* **Use a tiny record per aggregate**: `CategoryId`, `UserId`, …
* **Domain & repos speak typed IDs**; entities can keep primitives.
* **MapStruct/Converters** bridge the gap.
* Gains: **safety**, **clarity**, **future-proofing**.
* Cost: a 5-line record. Totally worth it.

---

## Typed ID — Q&A Cheatsheet

**Context:** Spring layered app (`domain/app/infra/web`), Java 17+, JPA/Hibernate, MapStruct
**Goal:** Understand why, when, and how to wrap IDs as records (e.g., `CategoryId`, `UserId`).

---

### Q: What’s a “typed ID”?

A tiny wrapper class (often a `record`) around a primitive identifier, e.g.:

```java
public record CategoryId(Long value) { }
```

It turns a generic number like `123L` into a **semantic, type-safe key** that the compiler can distinguish from any other ID.

---

### Q: Why bother wrapping something as simple as a `Long`?

Because raw primitives are easy to mix up.

```java
void deleteCategory(Long id);
void deleteUser(Long id);
```

A single typo can delete the wrong thing.
With typed IDs:

```java
void deleteCategory(CategoryId id);
void deleteUser(UserId id);
```

the compiler protects you.
It also makes signatures self-documenting and refactor-friendly.

---

### Q: Is it only about type safety?

No — it also gives you:

* **One place for validation** (`id > 0`, not null)
* **Future-proofing** (switch from `Long` → `UUID` without breaking 200 method signatures)
* **Consistent formatting** (string vs numeric representation)
* **Expressive domain language** (`CategoryId`, `OrderId`, etc.)

---

### Q: Where should I put the ID class?

Inside the same **domain package** as its aggregate.

```
domain/category/
  CategoryId.java
  Category.java
```

If multiple aggregates share an identical pattern, you can later promote to `domain/common/`, but start local — it keeps each domain self-contained.

---

### Q: When do I actually need a typed ID?

Use it when:

* The system has multiple entity types (users, categories, products).
* You want to avoid passing the wrong ID to the wrong repository.
* You might change key type later (e.g., `Long` → `UUID`).
* You want clean, self-documenting APIs.

Skip it when:

* You’re writing a tiny prototype or one-table script.
* You know the key type will *never* change and you don’t mind mixing IDs.

---

### Q: What’s inside a typed ID?

Usually just a single `value` field and a simple constructor check:

```java
public record CategoryId(Long value) {
  public CategoryId {
    if (value == null || value <= 0) throw new IllegalArgumentException("id.invalid");
  }
  public static CategoryId of(long v) { return new CategoryId(v); }
}
```

For UUIDs:

```java
public record CategoryId(UUID value) {
  public static CategoryId newId() { return new CategoryId(UUID.randomUUID()); }
}
```

---

### Q: How do I store it in the database?

Your JPA entity can still use a primitive field:

```java
@Id @GeneratedValue(strategy = IDENTITY)
private Long id;
```

Then use a **converter or MapStruct mapper** to unwrap/wrap the value:

```java
default CategoryId toCategoryId(Long id) { return id == null ? null : new CategoryId(id); }
default Long fromCategoryId(CategoryId id) { return id == null ? null : id.value(); }
```

This keeps the domain clean and the persistence layer pragmatic.

---

### Q: Does the domain ever generate IDs itself?

Usually **no** — persistence assigns IDs.
But for UUID/ULID, the domain can generate:

```java
public static CategoryId newId() { return new CategoryId(UUID.randomUUID()); }
```

This is handy when you want to create aggregates before saving them.

---

### Q: How does this work with `rehydrate(...)`?

When you rebuild a domain object from persistence, wrap the raw ID:

```java
Category.rehydrate(CategoryId.of(e.getId()), e.getName(), e.getSlug(), e.getCreatedAt());
```

The type ensures you can’t accidentally pass a mismatched ID.

---

### Q: What about JSON and API responses?

Return the plain scalar value (`Long` or `String`) in your DTOs, not the record itself.

```java
public record CategoryResponse(Long id, String name) {}
```

The typed ID is a *domain concern*, not a *transport concern*.

---

### Q: How do I write repository interfaces?

Use typed IDs everywhere:

```java
public interface CategoryRepository {
  Category save(Category category);
  Optional<Category> findById(CategoryId id);
  boolean existsById(CategoryId id);
}
```

Your infra adapter bridges between typed IDs and primitives.

---

### Q: What happens if I switch from Long to UUID later?

If you used typed IDs:

* Update only the record’s internal type and converters.
* Recompile — all callers still pass `CategoryId`.

If you didn’t:

* Change every repository, service, controller, and test signature.
* Cry.

---

### Q: Are typed IDs bad for performance?

No.
They compile down to lightweight objects or value records.
Any overhead is negligible next to a DB round-trip.

---

### Q: Do I need one converter per ID class?

Not necessarily. You can:

* Write one per aggregate (simple, explicit).
* Or write a generic `BaseIdConverter<ID extends BaseId<?>>` if you like abstraction.
  Most teams just copy-paste small ones for clarity.

---

### Q: Can I use Lombok for this?

No need — `record` already generates everything you’d want.
Lombok would just add noise here.

---

### Q: How do I test typed IDs?

**Unit tests:**

```java
assertThrows(IllegalArgumentException.class, () -> new CategoryId(0));
assertEquals(123L, new CategoryId(123L).value());
```

**Repo tests:**

* Ensure save/load preserves ID equality.
  **Mapper tests:**
* `CategoryId.of(entity.getId())` maps cleanly.

---

### Q: Common mistakes?

* Mixing typed and raw IDs in the same API (inconsistent).
* Serializing the record directly as JSON (clients don’t need wrapper objects).
* Forgetting to validate null/zero in constructors.
* Using `@Data` or `@AllArgsConstructor` on domain entities (bypasses rules).
* Making a single “GenericId” for everything (defeats the purpose).

---

### Q: TL;DR

> “Typed IDs turn opaque numbers into meaningful, safe keys.”
> Use them wherever identity matters,
> skip them only when you truly don’t care about safety or future changes.
