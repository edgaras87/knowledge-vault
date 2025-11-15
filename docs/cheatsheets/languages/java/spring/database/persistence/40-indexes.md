---
title: '@Index'
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@Index` annotation, detailing its attributes and usage for defining database indexes on entity tables.
aliases:
    - Jakarta Persistence — @Index Cheatsheet

---

# Jakarta Persistence — `@Index` Cheatsheet

Package: `jakarta.persistence.Index`
Used inside: `@Table(indexes = {})`

`@Index` defines a **database index** (optionally unique) on one or more columns of the table.
JPA doesn’t create indexes by itself unless you declare them here (or via migrations).

Indexes make lookups fast and uniqueness guaranteed (when `unique = true`).

---

## 0) Contents

| Section                             | Description                      | Link                                      |
| ----------------------------------- | -------------------------------- | ----------------------------------------- |
| **1) All Parameters**               | Full signature of @Index         | → [All Parameters](#1-all-parameters)     |
| **2) Concept & Mental Model**       | What indexes really do (DB view) | → [Concept](#2-concept--mental-model)     |
| **3) Simple Single-Column Index**   | Quick example                    | → [Single Column](#3-single-column-index) |
| **4) Unique Index (Business Keys)** | Enforcing uniqueness             | → [Unique Index](#4-unique-index)         |
| **5) Composite Index**              | Index on multiple columns        | → [Composite Index](#5-composite-index)   |
| **6) Index Naming Strategy**        | Why naming matters               | → [Naming](#6-index-naming-strategy)      |
| **7) Practical Patterns**           | Ready-to-use common patterns     | → [Patterns](#7-practical-patterns)       |
| **8) Best Practices**               | What to memorize                 | → [Best Practices](#8-best-practices)     |

---

## 1) All Parameters

Full `@Index` definition:

```java
@Index(
    name = "",
    columnList = "",
    unique = false
)
```

Meaning:

* **`name`** — index (or constraint) name in DB
* **`columnList`** — comma-separated list of columns
* **`unique`** — enforces uniqueness if true

Used inside `@Table`:

```java
@Table(
  name = "categories",
  indexes = {
    @Index(name = "uk_categories_slug", columnList = "slug", unique = true)
  }
)
```

---

## 2) Concept & Mental Model

Think of an index as:

> A sorted lookup structure that makes finding rows by a column extremely fast.

Without an index:

* DB scans entire table (“full table scan”).
* Fine for 20 rows, awful for 200 000 rows.

With an index:

* DB narrows down instantly to matching rows.
* Like having a book with a proper index at the end.

Indexes are **always** physical DB structures.

JPA only declares them — migration tools (Flyway/Liquibase) create them.

---

## 3) Single-Column Index

Example:

```java
@Entity
@Table(
  name = "products",
  indexes = @Index(
    name = "ix_products_sku",
    columnList = "sku"
  )
)
public class ProductEntity { ... }
```

Creates:

```sql
CREATE INDEX ix_products_sku ON products(sku);
```

Useful for fast lookups:

```sql
SELECT * FROM products WHERE sku = 'ABC-123';
```

Without index → full scan.
With index → instant jump to matching row(s).

---

## 4) Unique Index

To enforce business uniqueness (slugs, emails):

```java
@Table(
  name = "categories",
  indexes = @Index(
    name = "uk_categories_slug",
    columnList = "slug",
    unique = true
  )
)
```

Enforces:

* slug must be unique
* DB prevents duplicates (even under race conditions)

SQL equivalent:

```sql
ALTER TABLE categories
  ADD CONSTRAINT uk_categories_slug UNIQUE(slug);
```

Uniqueness at DB-level is the real guarantee.
App checks (`existsBySlug`) are only pre-flight.

---

## 5) Composite Index

Index on *multiple* columns.

Useful when queries filter by more than one column.

Example:

```java
@Table(
  name = "orders",
  indexes = {
    @Index(
      name = "ix_orders_customer_date",
      columnList = "customer_id, created_at"
    )
  }
)
public class OrderEntity { ... }
```

DB:

```sql
CREATE INDEX ix_orders_customer_date
  ON orders(customer_id, created_at);
```

Important ordering rule:

> Index is most effective for filtering by the **leading** column(s).

So indexing `(customer_id, created_at)` helps queries like:

```sql
WHERE customer_id = ? AND created_at > ?
WHERE customer_id = ?
```

But not:

```sql
WHERE created_at > ?   -- ignores the index
```

---

## 6) Index Naming Strategy

Choose a predictable naming scheme:

* `ix_<table>_<column>` for normal indexes
* `uk_<table>_<column>` for unique indexes
* `ix_<table>_<col1>_<col2>` for composite

Examples:

* `ix_products_sku`
* `uk_users_email`
* `ix_orders_customer_created_at`

Predictable names make migrations and debugging easier.

---

## 7) Practical Patterns

### 7.1 Unique business keys

```java
@Index(name = "uk_users_email", columnList = "email", unique = true)
```

Perfect for:

* emails
* usernames
* slugs
* external IDs

### 7.2 Foreign key performance

Always index FK columns:

```java
-- in migration, but conceptually:
INDEX ix_products_category_id (category_id)
```

JPA doesn't automatically create FK indexes — but you should.

### 7.3 Soft-delete pattern

```java
@Index(
  name = "ix_products_active",
  columnList = "deleted_at"
)
```

Speeds up “find active” queries.

### 7.4 Filtering + ordering

```java
@Index(
  name = "ix_orders_customer_created_at",
  columnList = "customer_id, created_at"
)
```

Supports:

* find orders by customer
* sort by date
* paginate efficiently

---

## 8) Best Practices

### 1. **Index all UNIQUE constraints**

If something must be unique, create a unique index.

### 2. **Index all FK columns**

Queries repeatedly filter by foreign key.

### 3. **Use composite indexes only when queries need them**

Don't index every combination “just in case”.

### 4. **Avoid indexing boolean columns**

Low selectivity → bad performance.

### 5. **Don't over-index**

Every index costs:

* disk space
* slower inserts/updates
* maintenance overhead

Always index intentionally.

### 6. **Let JPA annotate — but use migrations to enforce**

In a mature project:

* JPA: describes mapping
* Flyway/Liquibase: creates real indexes


