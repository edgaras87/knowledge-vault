---
title: '@Table'
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@Table` annotation, detailing its attributes and usage for mapping entities to database tables.
aliases:
    - Jakarta Persistence — @Table Cheatsheet

---

# Jakarta Persistence — `@Table` Cheatsheet

Package: `jakarta.persistence.Table`

`@Table` controls **how your entity maps to a database table**:
table name, schema/catalog, and database-level constraints like unique keys & indexes.

---

## 0) Contents

| Section                        | Description                                      | Link                                                 |
| ------------------------------ | ------------------------------------------------ | ---------------------------------------------------- |
| **1) All Parameters**          | Full list of `@Table` attributes                 | → [All Parameters](#1-all-parameters)                |
| **2) Explanation & Examples**  | What each parameter does, why, and how to use it | → [Explanation & Examples](#2-explanation--examples) |
| **2.1 `name`**                 | Override default table name                      | → [name](#21-name)                                   |
| **2.2 `schema`**               | Schema name for DBs like Postgres                | → [schema](#22-schema)                               |
| **2.3 `catalog`**              | Catalog name (rarely needed)                     | → [catalog](#23-catalog)                             |
| **2.4 `uniqueConstraints`**    | Old-school unique constraints                    | → [uniqueConstraints](#24-uniqueconstraints)         |
| **2.5 `indexes`**              | Modern index definitions with `@Index`           | → [indexes](#25-indexes)                             |
| **3) Full Practical Examples** | Realistic `@Table` usage patterns                | → [Practical Examples](#3-practical-examples)        |
| **4) Best Practices**          | What you should memorize                         | → [Best Practices](#4-best-practices)                |

---

## 1) All Parameters

The full annotation signature:

```java
@Table(
    name = "",
    catalog = "",
    schema = "",
    uniqueConstraints = {},
    indexes = {}
)
```

* **`name`** → table name
* **`schema`** → schema inside the database
* **`catalog`** → catalog name (multi-database servers)
* **`uniqueConstraints`** → array of `@UniqueConstraint`
* **`indexes`** → array of `@Index` (preferred modern method)

---

## 2) Explanation & Examples

---

### 2.1 `name`

Override the table name.
Default comes from class name → `CategoryEntity` → `category_entity`.

```java
@Entity
@Table(name = "categories")
public class CategoryEntity {}
```

Why use it?

* consistent naming (snake_case)
* avoid auto-generated weird names
* map Java class to existing table

---

### 2.2 `schema`

The schema your table lives in.

Postgres defaults to `public`, but you might have:

* `auth.users`
* `shop.products`
* `analytics.events`

```java
@Entity
@Table(name = "categories", schema = "shop")
public class CategoryEntity {}
```

Generated SQL:

```sql
CREATE TABLE shop.categories ( ... );
```

Useful when you split tables across schemas.

---

### 2.3 `catalog`

Catalog is like a *database group* in multi-DB environments (MySQL rarely uses it, Postgres never does).

Example (rare):

```java
@Entity
@Table(name = "users", catalog = "main_db")
public class UserEntity {}
```

You almost never need it unless DBA tells you.

---

### 2.4 `uniqueConstraints`

Older-style unique constraints.

```java
@Table(
  name = "users",
  uniqueConstraints = @UniqueConstraint(
    name = "uk_users_email",
    columnNames = {"email"}
  )
)
```

Equivalent DB constraint:

```sql
ALTER TABLE users
ADD CONSTRAINT uk_users_email UNIQUE (email);
```

**BUT:**
Modern code prefers `@Index(unique = true)` because it:

* is simpler
* is unified with index creation
* works reliably across providers

Still valid for composite keys:

```java
@Table(
  name = "orders",
  uniqueConstraints = @UniqueConstraint(
    name = "uk_order_user_product",
    columnNames = {"user_id", "product_id"}
  )
)
```

---

### 2.5 `indexes`

Modern indexing mechanism (recommended).

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

This produces:

```sql
CREATE UNIQUE INDEX uk_categories_slug ON categories(slug);
```

You can define multiple:

```java
@Table(
  name = "products",
  indexes = {
    @Index(name = "idx_products_category", columnList = "category_id"),
    @Index(name = "uk_products_sku", columnList = "sku", unique = true)
  }
)
```

---

## 3) Practical Examples

---

### 3.1 Basic table mapping

```java
@Entity
@Table(name = "categories")
public class CategoryEntity { ... }
```

---

### 3.2 Table with unique index (slug)

```java
@Entity
@Table(
  name = "categories",
  indexes = @Index(
    name = "uk_categories_slug",
    columnList = "slug",
    unique = true
  )
)
public class CategoryEntity { ... }
```

---

### 3.3 Composite unique constraint (old-style)

```java
@Entity
@Table(
  name = "order_items",
  uniqueConstraints = @UniqueConstraint(
    name = "uk_order_items_order_product",
    columnNames = {"order_id", "product_id"}
  )
)
public class OrderItemEntity {}
```

---

### 3.4 Named schema + indexes

```java
@Entity
@Table(
  name = "users",
  schema = "auth",
  indexes = {
    @Index(name = "uk_users_email", columnList = "email", unique = true),
    @Index(name = "idx_users_created_at", columnList = "created_at")
  }
)
public class UserEntity {}
```

SQL outcome:

```sql
CREATE TABLE auth.users ( ... );
CREATE UNIQUE INDEX uk_users_email ON auth.users(email);
CREATE INDEX idx_users_created_at ON auth.users(created_at);
```

---

## 4) Best Practices

* **Always** specify table name explicitly:

  ```java
  @Table(name = "categories")
  ```

* Prefer:

  ```java
  indexes = @Index(unique = true)
  ```

  instead of:

  ```java
  uniqueConstraints
  ```

* Use schemas only when needed (Postgres multi-schema setup).

* Use composite unique constraints sparingly — often better handled by unique index.

* Keep index names short but meaningful:

  * `uk_categories_slug`
  * `idx_products_category`

* Avoid `catalog` unless DBA forces it.

