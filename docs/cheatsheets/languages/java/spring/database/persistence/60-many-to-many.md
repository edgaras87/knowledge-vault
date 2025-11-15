---
title: '@ManyToMany' 
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@ManyToMany` annotation, detailing its purpose, parameters, and best practices for mapping many-to-many associations between entities in JPA.
aliases:
  - Jakarta Persistence — @ManyToMany Cheatsheet

---



# Jakarta Persistence — `@ManyToMany` Cheatsheet

Package: `jakarta.persistence.ManyToMany`

`@ManyToMany` maps a **many-to-many association** between two entities:

* one `Product` can have many `Tag`s
* one `Tag` can belong to many `Product`s

Under the hood this is **always** a join table.

In real projects you often replace it with an explicit join entity, but it’s important to understand what JPA does for you.

---

## 0) Contents

| Section                                     | Description                                   | Link                                                    |
| ------------------------------------------- | --------------------------------------------- | ------------------------------------------------------- |
| **1) All Parameters**                       | Full `@ManyToMany` signature                  | → [All Parameters](#1-all-parameters)                   |
| **2) Concept & Mental Model**               | What Many-to-Many really is in DB terms       | → [Concept](#2-concept--mental-model)                   |
| **3) Unidirectional `@ManyToMany`**         | One-sided mapping with `@JoinTable`           | → [Unidirectional](#3-unidirectional-manytomany)        |
| **4) Bidirectional `@ManyToMany`**          | Owning vs inverse side with `mappedBy`        | → [Bidirectional](#4-bidirectional-manytomany)          |
| **5) `@JoinTable` + `@JoinColumn` details** | How join table columns are configured         | → [JoinTable Details](#5-jointable--joincolumn-details) |
| **6) When NOT to use `@ManyToMany`**        | Why explicit join entities are usually better | → [When NOT to use](#6-when-not-to-use-manytomany)      |
| **7) Practical Patterns**                   | Copy-paste-ready examples                     | → [Practical Patterns](#7-practical-patterns)           |

---

## 1) All Parameters

Signature:

```java
@ManyToMany(
    mappedBy = "",
    cascade = {},
    fetch = FetchType.LAZY,   // default
    targetEntity = void.class
)
```

* **`mappedBy`** — for the **inverse** side (points to field name on owning side)
* **`cascade`** — cascade operations (PERSIST, MERGE, etc.)
* **`fetch`** — `LAZY` (default) or `EAGER`
* **`targetEntity`** — for generics/legacy use (almost never needed in modern code)

You almost always deal with:

* `mappedBy`
* sometimes `cascade`
* **never** `fetch = EAGER` unless you enjoy N+1 hell

---

## 2) Concept & Mental Model

In **relational DB**, many-to-many is **always**:

* table A (e.g. `products`)
* table B (e.g. `tags`)
* join table C (e.g. `product_tags`)

Join table typically has:

* FK to A (`product_id`)
* FK to B (`tag_id`)
* combined unique constraint (`product_id, tag_id`)

Example DB:

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(200) NOT NULL
);

CREATE TABLE tags (
  id BIGINT PRIMARY KEY,
  name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE product_tags (
  product_id BIGINT NOT NULL,
  tag_id BIGINT NOT NULL,
  PRIMARY KEY (product_id, tag_id),
  FOREIGN KEY (product_id) REFERENCES products(id),
  FOREIGN KEY (tag_id) REFERENCES tags(id)
);
```

`@ManyToMany` is just a **convenience wrapper** over this pattern.

---

## 3) Unidirectional `ManyToMany`

One side knows about the other, but not vice versa.

Example: `Product` has tags; `Tag` doesn’t know about products.

```java
@Entity
@Table(name = "tags")
public class TagEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = true, length = 50)
  private String name;
}
```

```java
@Entity
@Table(name = "products")
public class ProductEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 200)
  private String name;

  @ManyToMany
  @JoinTable(
    name = "product_tags",
    joinColumns = @JoinColumn(name = "product_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id")
  )
  private java.util.Set<TagEntity> tags = new java.util.HashSet<>();
}
```

* `@ManyToMany` on `ProductEntity.tags`
* `@JoinTable` defines join table + both FKs
* No `mappedBy` here → this is the **owning side**

This is the simplest ManyToMany mapping.

---

## 4) Bidirectional `ManyToMany`

Both ends have a collection to the other.
One side is **owning**, the other is **inverse**.

Owning side (defines `@JoinTable`):

```java
@Entity
@Table(name = "products")
public class ProductEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 200)
  private String name;

  @ManyToMany
  @JoinTable(
    name = "product_tags",
    joinColumns = @JoinColumn(name = "product_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id")
  )
  private java.util.Set<TagEntity> tags = new java.util.HashSet<>();
}
```

Inverse side (uses `mappedBy`):

```java
@Entity
@Table(name = "tags")
public class TagEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = true, length = 50)
  private String name;

  @ManyToMany(mappedBy = "tags")
  private java.util.Set<ProductEntity> products = new java.util.HashSet<>();
}
```

Key detail:

```java
@ManyToMany(mappedBy = "tags")
private Set<ProductEntity> products;
```

* `mappedBy = "tags"` → **field name** on owning side (`ProductEntity.tags`)
* Inverse side does **not** define `@JoinTable`

**Owning side**:

* writes join table rows
* controls persistence of association

**Inverse side**:

* is just a mirror
* changing its collection without updating owning side might be ignored

---

## 5) `@JoinTable` & `@JoinColumn` Details

`@JoinTable` has this shape:

```java
@JoinTable(
  name = "product_tags",
  joinColumns = @JoinColumn(name = "product_id"),
  inverseJoinColumns = @JoinColumn(name = "tag_id")
)
```

* `name` — join table name
* `joinColumns` — FKs pointing to **owning entity** (here: `product_id` → `products.id`)
* `inverseJoinColumns` — FKs pointing to **inverse entity** (here: `tag_id` → `tags.id`)

You can customize:

```java
@JoinTable(
  name = "product_tags",
  joinColumns = @JoinColumn(
    name = "product_id",
    referencedColumnName = "id"
  ),
  inverseJoinColumns = @JoinColumn(
    name = "tag_id",
    referencedColumnName = "id"
  )
)
```

You almost never need `referencedColumnName` unless you use non-standard PK columns.

---

## 6) When NOT to use `ManyToMany`

Most **real** systems avoid `@ManyToMany` and use an explicit join entity.

Why?

1. You often need extra data in the association, e.g.:

   * when product was tagged
   * who added the tag
   * ordering / weight
   * soft-delete flags

2. You get more control over:

   * cascading
   * lifecycle
   * constraints
   * querying

3. It makes things explicit and easier to debug.

---

### 6.1 Explicit join entity instead of `@ManyToMany`

Instead of:

```java
@ManyToMany
@JoinTable(name = "product_tags", ...)
Set<TagEntity> tags;
```

Do:

```java
@Entity
@Table(name = "product_tags")
public class ProductTagEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @ManyToOne(optional = false, fetch = FetchType.LAZY)
  @JoinColumn(name = "product_id", nullable = false)
  private ProductEntity product;

  @ManyToOne(optional = false, fetch = FetchType.LAZY)
  @JoinColumn(name = "tag_id", nullable = false)
  private TagEntity tag;

  @Column(name = "created_at", nullable = false)
  private java.time.Instant createdAt;
}
```

Now:

* You can query `ProductTagEntity` directly.
* You can attach extra columns.
* You keep clear control over cardinalities.

This is the pattern that scales better.

---

## 7) Practical Patterns

---

### 7.1 Small toy app / learning project — unidirectional

```java
@Entity
@Table(name = "products")
public class ProductEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 200)
  private String name;

  @ManyToMany
  @JoinTable(
    name = "product_tags",
    joinColumns = @JoinColumn(name = "product_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id")
  )
  private Set<TagEntity> tags = new HashSet<>();
}
```

Good for learning, PoCs, internal tools.

---

### 7.2 Bidirectional mapping (for demo / read API)

```java
@Entity
@Table(name = "products")
public class ProductEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @ManyToMany
  @JoinTable(
    name = "product_tags",
    joinColumns = @JoinColumn(name = "product_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id")
  )
  private Set<TagEntity> tags = new HashSet<>();
}
```

```java
@Entity
@Table(name = "tags")
public class TagEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @ManyToMany(mappedBy = "tags")
  private Set<ProductEntity> products = new HashSet<>();
}
```

Remember: owning side = `ProductEntity.tags`.

---

### 7.3 “Real-world” pattern — explicit join entity

**Recommended once projects get serious.**

```java
@Entity
@Table(
  name = "product_tags",
  indexes = {
    @Index(name = "uk_product_tags_product_tag", columnList = "product_id, tag_id", unique = true)
  }
)
public class ProductTagEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @ManyToOne(optional = false, fetch = FetchType.LAZY)
  @JoinColumn(name = "product_id", nullable = false)
  private ProductEntity product;

  @ManyToOne(optional = false, fetch = FetchType.LAZY)
  @JoinColumn(name = "tag_id", nullable = false)
  private TagEntity tag;

  @Column(name = "created_at", nullable = false)
  private Instant createdAt;
}
```

Then in `ProductEntity`:

```java
@OneToMany(mappedBy = "product")
private Set<ProductTagEntity> productTags = new HashSet<>();
```

And in `TagEntity`:

```java
@OneToMany(mappedBy = "tag")
private Set<ProductTagEntity> productTags = new HashSet<>();
```

Now you have full control, at the cost of a bit more code.

---

### 7.4 Fetch strategy

Never do this casually:

```java
@ManyToMany(fetch = FetchType.EAGER)
```

You will:

* load huge collections accidentally
* trigger N+1 queries
* cry over performance

Default `LAZY` is what you want.
Then you control fetching via queries (e.g., Spring Data `@EntityGraph` or explicit queries later).

---

### 7.5 DTO design

Even if you use `@ManyToMany` in entities, avoid exposing entity graphs directly in JSON. Instead:

* Flatten tags into simple DTOs:

```java
public record TagSummary(Long id, String name) {}
public record ProductDetail(Long id, String name, List<TagSummary> tags) {}
```

So persistence complexity stays hidden behind DTO mapping.

---

## Summary mental rule for `@ManyToMany`

* Great for **learning**, **small internal tools**, and **simple demos**.
* For serious domains:

    * understand it
    * then usually replace it with an explicit **join entity** + two `@ManyToOne`s.

