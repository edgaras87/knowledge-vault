---
title: '@Entity'
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@Entity` annotation, detailing its purpose, attributes, and usage for defining persistent entities in JPA.
aliases:
  - Jakarta Persistence — @Entity Cheatsheet

---

# Jakarta Persistence — `@Entity` Cheatsheet

Package: `jakarta.persistence.Entity`

`@Entity` marks a **persistent domain type** that JPA/Hibernate tracks and maps to a table (or view).

No `@Entity` → no persistence, no dirty checking, no relationships, nothing.
This is the entry ticket.

---

## 0) Contents

| Section                             | Description                                 | Link                                                    |
| ----------------------------------- | ------------------------------------------- | ------------------------------------------------------- |
| **1) Annotation Basics**            | Signature, where it lives                   | → [Basics](#1-annotation-basics)                        |
| **2) What Makes a Class an Entity** | Rules & requirements                        | → [Entity Definition](#2-what-makes-a-class-an-entity)  |
| **3) Table Mapping Flow**           | How `@Entity` + others form the mapping     | → [Mapping Flow](#3-table-mapping-flow)                 |
| **4) Examples**                     | Minimal, typical, and Lombok-style entities | → [Examples](#4-examples)                               |
| **5) Entity Lifecycle & Identity**  | Persistence context, equals/hashCode        | → [Lifecycle & Identity](#5-entity-lifecycle--identity) |
| **6) What is *not* an Entity**      | Value objects, DTOs, projections            | → [Not Entities](#6-what-is-not-an-entity)              |
| **7) Best Practices**               | Things to hard-wire into your habits        | → [Best Practices](#7-best-practices)                   |

---

## 1) Annotation Basics

```java
import jakarta.persistence.Entity;

@Entity
public class CategoryEntity {
    // ...
}
```

Signature:

```java
@Entity(
    name = ""  // optional, custom entity name
)
```

* **`name`** — JPA “entity name” used in JPQL/HQL (defaults to class name).
  It **does not** control the table name — that’s `@Table(name = "…")`.

In practice, you almost never set `name`; `@Table` is what you care about.

---

## 2) What Makes a Class an Entity

Minimum requirements for a valid entity:

1. Annotated with `@Entity`
2. Has a primary key field annotated with `@Id`
3. Has a no-arg constructor (at least protected)
4. Is a top-level class (no local/anonymous classes)
5. Not `final` if proxies are used (Hibernate likes to subclass)

Example of a valid minimal entity:

```java
@Entity
public class CategoryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;

  protected CategoryEntity() {
    // JPA needs this
  }

  public CategoryEntity(String name) {
    this.name = name;
  }
}
```

If you forget `@Id`, Hibernate will complain at startup:

> No identifier specified for entity ...

---

## 3) Table Mapping Flow

`@Entity` doesn’t know about tables by itself. It works together with:

* `@Table` → which table
* `@Column` → which columns
* `@Id` / `@GeneratedValue` → primary key
* `@OneToMany`, `@ManyToOne`, etc. → relationships

Example:

```java
@Entity                      // class is an entity
@Table(name = "categories")  // mapped to table "categories"
public class CategoryEntity {

  @Id                        // primary key
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 100)
  private String name;

  @Column(nullable = false, length = 100, unique = true)
  private String slug;
}
```

Rough mental mapping:

* `@Entity` → “Hibernate, please manage this class.”
* `@Table` → “Use this table.”
* `@Id` → “This field is the primary key.”
* `@Column` → “These are the columns and constraints.”
* Relationship annotations → “Here’s how tables are linked.”

---

## 4) Examples

---

### 4.1 Minimal entity (no `@Table`, default mapping)

```java
@Entity
public class TagEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;
}
```

Hibernate will derive:

* table name from class name (`tag_entity` or `tagentity` depending on naming strategy)
* columns from fields

You usually want to be more explicit.

---

### 4.2 Typical Spring/JPA entity (with `@Table`)

```java
@Entity
@Table(name = "categories")
public class CategoryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 100)
  private String name;

  @Column(nullable = false, length = 100, unique = true)
  private String slug;
}
```

This is basically how you’ll write most of your entities.

---

### 4.3 Entity with Lombok (common pattern)

```java
@Entity
@Table(name = "categories")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CategoryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 100)
  private String name;

  @Column(nullable = false, length = 100, unique = true)
  private String slug;

  public CategoryEntity(String name, String slug) {
    this.name = name;
    this.slug = slug;
  }
}
```

* `@NoArgsConstructor(PROTECTED)` → JPA can instantiate; you don’t expose it publicly.
* Manually define “real” constructor for your own use.

---

## 5) Entity Lifecycle & Identity

Entities aren’t just “rows”. JPA tracks them in a **persistence context**.

### 5.1 Lifecycle states (simplified)

* **New / transient** — not yet persisted (no ID, or not managed)
* **Managed** — attached to EntityManager / Session
* **Detached** — was managed, now disconnected
* **Removed** — scheduled for delete

Example flow:

```java
CategoryEntity category = new CategoryEntity("Books", "books"); // new

em.persist(category);   // managed + INSERT scheduled
category.setName("Books & Media"); // tracked change

em.flush();             // UPDATE if needed
em.remove(category);    // DELETE scheduled
```

### 5.2 Identity: `@Id` is the anchor

Within a persistence context:

```java
CategoryEntity c1 = em.find(CategoryEntity.class, 1L);
CategoryEntity c2 = em.find(CategoryEntity.class, 1L);

c1 == c2; // true (same instance)
```

Same entity type + same PK = same instance.

### 5.3 equals/hashCode

Don’t obsess yet, but the core rules:

* Don’t base equality on all fields
* Safest is ID-based equality once the ID is assigned
* Before ID exists, equality is tricky — let frameworks or your pattern handle it

For now: avoid writing fancy equals/hashCode on entities unless you really need them.

---

## 6) What is *not* an Entity

Important boundaries:

### 6.1 DTOs

* Records/classes used for **API input/output**
* No `@Entity`
* Live in `web` layer (`dto.request`, `dto.response` etc.)
* Often annotated with validation (`@NotBlank`, `@Size`)

```java
public record CreateCategoryRequest(
  String name,
  String slug
) {}
```

### 6.2 Value Objects / Embeddables

Small value types that don’t have their own table.

```java
@Embeddable
public class Money {
  @Column(precision = 10, scale = 2)
  private BigDecimal amount;

  @Column(length = 3)
  private String currency;
}
```

Used with `@Embedded` in an entity.
Still not full entities.

### 6.3 Projections / Views

Interface-based DTOs in Spring Data, or “read models” that slice data differently.
Not annotated with `@Entity`.

**Rule:**
If it has `@Entity`, it has its own table and identity.
If it doesn’t — it’s probably DTO/VO/projection.

---

## 7) Best Practices

### 1. Keep entities “persistence + domain”, not “web + persistence + everything”

Entities should:

* map to DB
* model core domain behavior

Entities should **not**:

* know about HTTP
* carry web validation for all layers
* be exposed directly in API responses (use DTOs)

### 2. Always be explicit with table name

```java
@Table(name = "categories")
```

Don’t rely on default naming strategy if you care about schema stability.

### 3. Always have a protected no-arg constructor

Even with Lombok:

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
```

### 4. Prefer surrogate primary keys (Long or UUID)

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

Keep business identifiers (slug, email, etc.) as separate unique columns.

### 5. Don’t stuff entities with everything

If you have tons of optional fields that only make sense in some context:

* consider splitting into separate entities with relationships
* or model them as embeddables

### 6. Keep relationships minimal and lazy

* Avoid giant graphs of bidirectional relationships
* Use `LAZY` fetching everywhere by default
* Map only what you actually use

