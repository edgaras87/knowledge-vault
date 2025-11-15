---
title: '@OneToMany' 
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@OneToMany` annotation, detailing its purpose, parameters, and best practices for mapping one-to-many relationships in JPA.
aliases:
  - Jakarta Persistence — @OneToMany Cheatsheet

---

# Jakarta Persistence — `@OneToMany` Cheatsheet

Package: `jakarta.persistence.OneToMany`

`@OneToMany` represents:

* **One parent → many children**,
* but *the parent does NOT own the foreign key*.
* The FK always lives in the **many** side.

This annotation is almost always the **inverse side** of a `@ManyToOne` mapping.
Think of it as *“reflection-only”* unless configured with a join table (rare, for legacy schemas).

---

## 0) Contents

| Section                                      | Description                   | Link                                                           |
| -------------------------------------------- | ----------------------------- | -------------------------------------------------------------- |
| **1) All Parameters**                        | Complete signature            | → [All Parameters](#1-all-parameters)                          |
| **2) Mental Model**                          | How @OneToMany really works   | → [Mental Model](#2-mental-model)                              |
| **3) Inverse-Side Mapping (the normal one)** | `mappedBy` pattern            | → [Inverse Side](#3-inverse-side---the-normal-pattern)         |
| **4) Owning-Side Join-Table Mapping**        | Rare, usually not recommended | → [Owning Side (Join Table)](#4-owning-side-join-table---rare) |
| **5) Cascade, Orphans, Lazy/Eager**          | Practical behavior controls   | → [Cascade & Orphans](#5-cascade-orphan-removal-fetch)         |
| **6) Examples**                              | Ready patterns                | → [Examples](#6-examples)                                      |
| **7) Best Practices**                        | What to memorize              | → [Best Practices](#7-best-practices)                          |

---

## 1) All Parameters

```java
@OneToMany(
    mappedBy = "",
    cascade = {},
    fetch = FetchType.LAZY,
    orphanRemoval = false,
    targetEntity = void.class
)
```

* **`mappedBy`** → inverse side field name on the child (most important)
* **`cascade`** → operations propagated to children
* **`fetch`** → always prefer LAZY (default)
* **`orphanRemoval`** → delete children removed from collection
* **`targetEntity`** → for generics/legacy (rarely used)

---

## 2) Mental Model

The **key** to `OneToMany`:

> `@OneToMany` does NOT own the relationship.
> The FK belongs to the `@ManyToOne` side.

So this:

```java
@OneToMany(mappedBy = "category")
private Set<ProductEntity> products;
```

does **not** create a column in the parent table.
The column always lives here:

```java
@ManyToOne
@JoinColumn(name = "category_id")
private CategoryEntity category;
```

You never put a FK column in the “one” side.
That’s why `mappedBy` is mandatory for the normal pattern.

---

## 3) Inverse Side — the Normal Pattern

This is the 99% real-world usage.

### Parent (Category) — inverse side:

```java
@Entity
@Table(name = "categories")
public class CategoryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 100)
  private String name;

  @OneToMany(mappedBy = "category", fetch = FetchType.LAZY)
  private Set<ProductEntity> products = new HashSet<>();
}
```

### Child (Product) — owning side:

```java
@Entity
@Table(name = "products")
public class ProductEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 200)
  private String name;

  @ManyToOne(optional = false, fetch = FetchType.LAZY)
  @JoinColumn(name = "category_id", nullable = false)
  private CategoryEntity category;
}
```

**`mappedBy = "category"`** means:

* “the FK lives in `ProductEntity.category`”
* Category just exposes a collection
* It does NOT control inserts/updates of the FK

---

## 4) Owning Side (Join Table) — Rare

You can make `@OneToMany` the owning side using a join table:

```java
@OneToMany
@JoinTable(
  name = "category_products",
  joinColumns = @JoinColumn(name = "category_id"),
  inverseJoinColumns = @JoinColumn(name = "product_id")
)
private Set<ProductEntity> products;
```

But this creates a **join table**, not a FK in the child table.

**Problems:**

* Harder to query
* Not the relational model you actually want
* You lose natural FK (`product.category_id`)
* ORM magic becomes weird

This style is discouraged unless maintaining legacy schema.

---

## 5) Cascade, Orphan Removal, Fetch

### Cascade

```java
@OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
```

Propagates changes to children. Use cautiously:

* `CascadeType.PERSIST` — common (save parent + children)
* `CascadeType.REMOVE` — deletes children when parent deleted
* `CascadeType.ALL` — includes above

### Orphan Removal

```java
@OneToMany(mappedBy = "category", orphanRemoval = true)
private List<ProductEntity> products;
```

If you remove a child from the collection:

```java
category.getProducts().remove(product);
```

→ JPA DELETEs it.

Useful for “child belongs entirely to parent” patterns.

Avoid in shared relationships.

### Lazy vs Eager

Always use LAZY:

```java
@OneToMany(fetch = FetchType.LAZY)
```

EAGER collections will explode your query performance:

* N+1 queries
* Huge unnecessary loads

---

## 6) Examples

---

### 6.1 Basic One-to-Many (Category → Products)

Parent:

```java
@OneToMany(mappedBy = "category", fetch = FetchType.LAZY)
private Set<ProductEntity> products = new HashSet<>();
```

Child:

```java
@ManyToOne(optional = false)
@JoinColumn(name = "category_id", nullable = false)
private CategoryEntity category;
```

DB schema (simplified):

```sql
products.category_id → categories.id
```

---

### 6.2 With cascade and orphan removal

Parent:

```java
@OneToMany(
  mappedBy = "order",
  cascade = CascadeType.ALL,
  orphanRemoval = true
)
private List<OrderItemEntity> items = new ArrayList<>();
```

Child:

```java
@ManyToOne(optional = false)
@JoinColumn(name = "order_id")
private OrderEntity order;
```

Now:

* Adding item → persisted automatically
* Removing item → DELETE from DB
* Deleting order → deletes items too

This fits DDD Aggregate patterns.

---

### 6.3 Bidirectional convenience helpers

To maintain both sides:

```java
public void addProduct(ProductEntity product) {
  products.add(product);
  product.setCategory(this);
}

public void removeProduct(ProductEntity product) {
  products.remove(product);
  product.setCategory(null);
}
```

This prevents inconsistent state where parent has child but child still points to old parent.

---

### 6.4 Unidirectional One-to-Many (Join Table) — legacy style

```java
@OneToMany
@JoinTable(
  name = "author_books",
  joinColumns = @JoinColumn(name = "author_id"),
  inverseJoinColumns = @JoinColumn(name = "book_id")
)
private Set<BookEntity> books;
```

This avoids FK in the child table, but creates a join table.
Better replaced with explicit:

* `@OneToMany(mappedBy...)`
* or simply model the relationship differently.

---

## 7) Best Practices

### 1. Use `@OneToMany(mappedBy = ...)` almost always

FK belongs on the **many** side.

### 2. Always LAZY

EAGER = performance landmine.

### 3. Don’t overuse cascade

Only cascade when the child should “belong to” the parent.

### 4. Use orphanRemoval only when appropriate

Great for Order → OrderItems
Bad for Category → Products (products shouldn’t be deleted)

### 5. Keep collections as `Set` unless order matters

* `Set` avoids duplicates naturally
* `List` is okay but watch out for order-sensitive diffs

### 6. Maintain both sides of bidirectional mappings

Or you'll get inconsistent state.

### 7. In complex domains:

Prefer explicit join entities over magical join tables.

