---
title: Slug
date: 2025-11-12
tags: 
  - spring
  - domain-driven-design
  - architecture
  - cheatsheet 
summary: A practical cheatsheet for implementing slugs in a Spring layered architecture using Java 17+, MapStruct, JPA, and Flyway. Covers slug definition, usage patterns, DB schema, and best practices.
aliases:
  - Spring Domain Layer - Slug Cheatsheet
---


---

# Slug — Practical Cheatsheet

**Path:** `docs/cheatsheets/domain/slug/index.md`
**Scope:** Spring layered (`web/app/domain/infra`), Java 17+, MapStruct, JPA, Flyway

---

## What a slug is (and isn’t)

* **Slug** = canonical, URL-safe, case-normalized key derived from human text. Example: `"Sci-Fi & Fantasy"` → `"sci-fi-fantasy"`.
* **Not** a display label. Keep `name` for humans; keep `slug` for machines.
* Common jobs:

  * **Uniqueness**: `unique(slug)` in DB (case/locale safe).
  * **Routing**: readable URLs like `/api/categories/sci-fi-fantasy`.
  * **Lookup**: `findBySlug(...)` is deterministic.

---

## Where it lives (project shape)

```
src/main/java/com/example/shop/
  domain/category/
    Slug.java            # value object
    Category.java        # uses Slug
  infra/jpa/category/
    CategoryEntity.java  # has 'slug' column
  infra/mapping/category/
    CategoryMap.java     # maps entity <-> domain
```

---

## Slug value object (copy-paste)

```java
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

  /** Derive canonical slug from user-facing name (creation/rename). */
  public static Slug of(String name) {
    String base = Objects.requireNonNull(name).trim().toLowerCase(Locale.ROOT);
    base = base.replaceAll("\\s+", "-").replaceAll("[^a-z0-9-]", "");
    base = base.replaceAll("(^-+|-+$)", "");
    return new Slug(base);
  }

  /** Wrap a canonical slug read from persistence (no re-normalization). */
  public static Slug ofSlug(String persisted) { return new Slug(persisted); }
}
```

---

## Domain usage pattern

* **Create/rename paths**: derive slug from `name` with `Slug.of(name)`.
* **Rehydrate (load from DB)**: wrap persisted slug with `Slug.ofSlug(dbValue)`; **do not** re-derive on read.

```java
public final class Category {
  // fields: id, String name, Slug slug, Instant createdAt
  private Category(/* ... */) { /* ... */ }

  public static Category createNew(String name, Instant now) {
    String n = requireNonBlank(name);
    return new Category(null, n, Slug.of(n), now);
  }

  public static Category rehydrate(CategoryId id, String name, String slug, Instant createdAt) {
    return new Category(id, name, Slug.ofSlug(slug), createdAt);
  }

  public Category rename(String newName) {
    String n = requireNonBlank(newName);
    // If policy = immutable slug, return this.slug; else re-derive:
    return new Category(this.id, n, Slug.of(n), this.createdAt);
  }
}
```

---

## DB schema (Flyway)

```sql
-- Vxxx__categories.sql
create table categories (
  id         bigserial primary key,
  name       varchar(100) not null,
  slug       varchar(100) not null,
  created_at timestamp not null default now(),
  unique (slug)
);
create index ix_categories_created_at on categories(created_at);
```

* Unique constraint on `slug`, not on `name`.
* Keep `name` free for cosmetic edits; `slug` is the anchor.

---

## JPA entity (infra)

```java
@Entity
@Table(name="categories",
       indexes=@Index(name="uk_categories_slug", columnList="slug", unique=true))
@Getter @Setter @NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CategoryEntity {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable=false, length=100) private String name;
  @Column(nullable=false, length=100) private String slug;
  @Column(name="created_at", nullable=false) private Instant createdAt;
}
```

---

## MapStruct mapping (entity ↔ domain)

```java
@Mapper(componentModel = "spring")
public interface CategoryMap {

  // Domain -> Entity (new save)
  @Mapping(target = "id", ignore = true)
  @Mapping(target = "slug", expression = "java(domain.slug())")
  CategoryEntity toEntityNew(Category domain);

  // Entity -> Domain (rehydration)
  default Category toDomain(CategoryEntity e) {
    return Category.rehydrate(
      CategoryId.of(e.getId()),
      e.getName(),
      e.getSlug(),            // already canonical
      e.getCreatedAt()
    );
  }
}
```

---

## Repository API (domain port)

```java
public interface CategoryRepository {
  boolean existsBySlug(Slug slug);
  Optional<Category> findBySlug(Slug slug);
  Category save(Category category);
}
```

Infra adapter converts `Slug` to `String` at the DB edge.

---

## Endpoint patterns

Choose your URL strategy (pick one and be consistent):

1. **Slug URLs (pretty, stable if slug immutable)**

```
GET  /api/categories/{slug}
POST /api/categories       # body: { "name": "Sci-Fi" } -> server derives slug
```

2. **ID URLs with cosmetic slug (super stable)**

```
GET /api/categories/{id}-{slug}
```

Back-end uses `id`; slug is for looks.

---

## Policy: should slug change on rename?

* **Immutable (recommended for public links):** keep slug fixed; rename only changes `name`.
  If you later decide to change slug, treat it as a special admin action and set up redirects/aliases.
* **Derived:** re-derive on rename; requires redirects to avoid broken links.

---

## Tests you actually want

**Domain unit**

* `createNew("  Sci Fi  ")` → slug `sci-fi`.
* `rehydrate(id,"Name","sci-fi",at)` → slug unchanged.
* `rename("Science Fiction")` → slug re-derived **only if policy**.

**Repo (`@DataJpaTest`)**

* Unique constraint on `slug` enforced.
* `existsBySlug`, `findBySlug` work.

**Web (`@WebMvcTest`)**

* POST creates slug from name; returns `201` + `Location`.
* Duplicate name (same slug) → `409` with ProblemDetail.

---

## When to use a slug (and when not)

Use for:

* Categories, tags, collections, blog/product titles, user handles.

Skip for:

* **Email** (use canonical email column instead),
* internal-only entities with no human facing identity,
* pure numeric/UUID keys where readable URLs aren’t needed.

---

## Migration/alias strategy (optional but handy)

If you allow slug changes later:

* Add `category_slug_aliases(category_id bigint, slug varchar(100), unique(category_id, slug))`.
* On lookup by slug:

  1. try `categories.slug = ?`
  2. else try `category_slug_aliases.slug = ?` → redirect to primary slug.
* This preserves old links after renames.

---

## Pitfalls to avoid

* Recomputing slug on **read** (drifts historic data).
* Unique constraint on `name` instead of `slug`.
* Mixing presentation logic into slug rules (slug is for machines).
* Locale-sensitive lowercasing; always use `Locale.ROOT`.

---

## TL;DR card

* Keep **`name`** for display, **`slug`** for keys/URLs.
* Derive slug on **create/rename**, never on **rehydrate**.
* `unique(slug)` in DB.
* Prefer **immutable slug** for stable links; add aliases if you must change it later.


---

## Q&A's

**Path:** `docs/cheatsheets/domain/slug/q-and-a.md`
**Context:** Spring layered app (`domain/app/infra/web`), Java 17+, JPA + MapStruct

---

### Q: What exactly is a slug?

A slug is a **canonical, URL-safe key** derived from a human name.
It’s lowercased, hyphenated, and stripped of unsafe characters — e.g.
`"Sci-Fi & Fantasy"` → `"sci-fi-fantasy"`.

It’s how you get readable URLs like `/api/categories/sci-fi-fantasy`
and consistent uniqueness without worrying about casing or accents.

---

### Q: Why not just lowercase the name and call it a day?

Lowercasing doesn’t make text URL-safe or consistent.
You still have to:

* trim whitespace
* replace spaces with `-`
* remove punctuation and accents
* enforce length/character rules

The slug object encapsulates all that normalization once, so you don’t re-implement it in five different layers.

---

### Q: How is `slug` different from `name`?

| Field  | Meaning                  | Mutable?   | Used for                    |
| ------ | ------------------------ | ---------- | --------------------------- |
| `name` | display label for humans | yes        | UI, translations, cosmetics |
| `slug` | canonical machine key    | usually no | uniqueness, URLs, lookups   |

Changing the `name` doesn’t have to break your links.

---

### Q: Should the slug change when I rename the category?

That’s a **policy choice**.

| Policy               | Behavior                             | URL stability                   | Typical use                     |
| -------------------- | ------------------------------------ | ------------------------------- | ------------------------------- |
| **Immutable slug**   | slug stays fixed; only name changes  | ✅ stable                        | public sites, SEO, shared links |
| **Derived slug**     | slug re-derived from new name        | ❌ links break unless redirected | internal admin tools            |
| **Hybrid (id-slug)** | `/42-sci-fi-fantasy`; id is real key | ✅ stable                        | e-commerce, hybrid URLs         |

Most public APIs freeze the slug after creation.

---

### Q: Why store both `name` and `slug` in the database?

They serve different audiences.
`name` can vary by language, capitalization, emoji, etc.
`slug` must remain canonical and unique.

You’ll thank yourself when marketing changes “Books” → “Books & Media”
and you don’t need to rewrite every product URL.

---

### Q: Where in code do we create vs. trust the slug?

* **Create/rename paths:** `Slug.of(name)` — derive & normalize.
* **Load from DB:** `Slug.ofSlug(persisted)` — wrap canonical value, no normalization.

This prevents historic data from “drifting” if normalization rules evolve.

---

### Q: What does `rehydrate(...)` have to do with slugs?

`rehydrate(...)` reconstructs a domain object from persisted state.
It uses `Slug.ofSlug(...)` to trust the stored canonical slug, instead of recomputing it.
That guarantees you reload *exactly* what you saved.

---

### Q: How do I enforce uniqueness?

Database constraint:

```sql
unique (slug)
```

and repository method:

```java
boolean existsBySlug(Slug slug);
```

That’s more reliable than comparing display names with locale-dependent case rules.

---

### Q: Is the slug only for endpoints?

Not only.
It’s used for:

* unique keys in the DB
* URL paths
* lookups and joins
* caching keys (`"category:sci-fi-fantasy"`)

So even if you never expose it publicly, it’s still valuable internally.

---

### Q: What about emails or usernames — should they be slugs?

* **Email:** no, just store a canonicalized version (trim + lowercase).
* **User handles / tags:** yes, perfect slug candidates (`@edgaras` → `"edgaras"`).
* **Full names:** usually not; keep as display labels.

---

### Q: How do I validate or limit slug characters?

In the value object:

```java
private static final Pattern ALLOWED = Pattern.compile("[a-z0-9-]+");
```

Throw `IllegalArgumentException` if it doesn’t match.
You can later extend this to transliterate accents or support Unicode.

---

### Q: Where should the slug live in the project?

Inside the **domain layer** as a value object (`Slug.java`).
The web, app, and infra layers use it but never re-implement normalization.

---

### Q: What happens if I change slug rules in version 2 of my app?

Old rows keep their old canonical slugs.
New records use the new rules.
You don’t rewrite history silently; that’s why `rehydrate` never re-derives.

---

### Q: Any performance or code-size penalty?

None worth mentioning.
A tiny `record` and one regex check per create/rename — negligible.

---

### Q: Common mistakes?

* Re-deriving slug on read (mutates old data).
* Unique constraint on `name` instead of `slug`.
* Locale-dependent lowercasing (`toLowerCase()` without `Locale.ROOT`).
* Using Lombok `@AllArgsConstructor` on domain classes (bypasses invariants).
* Treating slug as optional — it’s better to make it mandatory and validated once.

---

### Q: How do I test slug behavior?

**Unit:**

```java
Slug s = Slug.of("  Sci Fi  ");
assertEquals("sci-fi", s.value());
assertThrows(IllegalArgumentException.class, () -> new Slug("Bad!Value"));
```

**Repo:**

* Insert duplicates → expect constraint violation.
  **Web:**
* POST `/api/categories` → 201 + Location with derived slug.

---

### Q: TL;DR

> “Slug is the machine’s version of a name.”
> Derive once when creating, trust forever after.
> Humans edit names, machines stick to slugs.


