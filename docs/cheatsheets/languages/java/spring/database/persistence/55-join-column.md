---
title: '@JoinColumn'
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@JoinColumn` annotation, detailing its purpose, parameters, and best practices for mapping foreign key columns in entity relationships.
aliases:
    - Jakarta Persistence — @JoinColumn Cheatsheet

---

# Jakarta Persistence — `@JoinColumn` Cheatsheet

Package: `jakarta.persistence.JoinColumn`

`@JoinColumn` configures the **foreign key column** that links two tables in a relationship:

* `@ManyToOne`
* `@OneToOne`
* part of `@JoinTable` for `@ManyToMany`

Think of it as: *“this field is backed by that FK column in this table.”*

---

## 0) Contents

| Section                              | Description                         | Link                                                              |
| ------------------------------------ | ----------------------------------- | ----------------------------------------------------------------- |
| **1) All Parameters**                | Full `@JoinColumn` signature        | → [All Parameters](#1-all-parameters)                             |
| **2) Explanation & Examples**        | What each param does with code      | → [Explanation & Examples](#2-explanation--examples)              |
| **2.1 `name`**                       | FK column name in owning table      | → [name](#21-name--column-name)                                   |
| **2.2 `referencedColumnName`**       | Target PK/column in the other table | → [referencedColumnName](#22-referencedcolumnname--target-column) |
| **2.3 `nullable`**                   | Is FK allowed to be null?           | → [nullable](#23-nullable--required-vs-optional-relationship)     |
| **2.4 `unique`**                     | Enforce 1:1 via unique FK           | → [unique](#24-unique--enforcing-one-to-one)                      |
| **2.5 `insertable` / `updatable`**   | Read-only FKs, DB-managed joins     | → [insertable/updatable](#25-insertable--updatable)               |
| **2.6 `columnDefinition`**           | Custom SQL for FK column            | → [columnDefinition](#26-columndefinition)                        |
| **2.7 `table`**                      | Join column in secondary table      | → [table](#27-table)                                              |
| **2.8 `foreignKey`**                 | Customize foreign key constraint    | → [foreignKey](#28-foreignkey)                                    |
| **3) Typical Usages**                | ManyToOne, OneToOne, etc.           | → [Typical Usages](#3-typical-usages)                             |
| **4) Best Practices & Mental Model** | What to memorize                    | → [Best Practices](#4-best-practices--mental-model)               |

---

## 1) All Parameters

Full annotation:

```java
@JoinColumn(
    name = "",
    referencedColumnName = "",
    unique = false,
    nullable = true,
    insertable = true,
    updatable = true,
    columnDefinition = "",
    table = "",
    foreignKey = @ForeignKey(ConstraintMode.PROVIDER_DEFAULT)
)
```

* **`name`** — FK column name in *this* table
* **`referencedColumnName`** — column name in target table (usually PK)
* **`unique`** — create unique constraint on FK column
* **`nullable`** — can FK be null?
* **`insertable`/`updatable`** — whether JPA writes this column
* **`columnDefinition`** — raw SQL definition for FK column
* **`table`** — secondary table (rare)
* **`foreignKey`** — control foreign key constraint name/mode

---

## 2) Explanation & Examples

---

### 2.1 `name` — column name

**What it does:**
Name of the foreign key column in the **owning** table.

Typical `@ManyToOne`:

```java
@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "category_id", nullable = false)
private CategoryEntity category;
```

DB (products table):

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  category_id BIGINT NOT NULL,
  -- FK added via migration or foreignKey attr
);
```

If you omit `name`, JPA derives something like `category_id` or `category_id_id` depending on naming strategy. Explicit is better.

---

### 2.2 `referencedColumnName` — target column

**What it does:**
Which column in the **target** table this FK points to.

Default: primary key column of the target entity.

You rarely set this, but when you do:

```java
@Entity
@Table(name = "countries")
public class CountryEntity {

  @Id
  @Column(name = "code", length = 2)
  private String code; // "LT", "DE"
}
```

```java
@Entity
@Table(name = "addresses")
public class AddressEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @ManyToOne(optional = false)
  @JoinColumn(
    name = "country_code",
    referencedColumnName = "code"
  )
  private CountryEntity country;
}
```

Now `addresses.country_code` → `countries.code`.

If you use normal Long PKs, you almost never need `referencedColumnName`.

---

### 2.3 `nullable` — required vs optional relationship

**What it does:**
Controls whether FK column allows `NULL`.

Required relationship:

```java
@ManyToOne(optional = false, fetch = FetchType.LAZY)
@JoinColumn(name = "category_id", nullable = false)
private CategoryEntity category;
```

DB:

```sql
category_id BIGINT NOT NULL
```

Optional relationship:

```java
@ManyToOne(optional = true, fetch = FetchType.LAZY)
@JoinColumn(name = "parent_id", nullable = true)
private CategoryEntity parent;
```

DB:

```sql
parent_id BIGINT NULL
```

**Rule:**
Use both:

* `optional = false` on mapping
* `nullable = false` in `@JoinColumn`

for clarity and consistency.

---

### 2.4 `unique` — enforcing one-to-one

**What it does:**
Adds a **unique constraint** on FK column → effectively 1:1.

Example: profile per user.

```java
@Entity
@Table(name = "user_profiles")
public class UserProfileEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @OneToOne(fetch = FetchType.LAZY, optional = false)
  @JoinColumn(
    name = "user_id",
    nullable = false,
    unique = true
  )
  private UserEntity user;
}
```

DB constraint:

```sql
ALTER TABLE user_profiles
  ADD CONSTRAINT uk_user_profiles_user UNIQUE (user_id);
```

This ensures:

* A profile belongs to **exactly one** user
* Each user can have **at most one** profile

---

### 2.5 `insertable` / `updatable`

**What they do:**
Control whether JPA writes the FK column on INSERT/UPDATE.

Example: read-only relationship mapping a view or DB-managed link.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(
  name = "category_id",
  insertable = false,
  updatable = false
)
private CategoryEntity category;
```

Effect:

* JPA reads `category_id`
* But **never sets or changes it**
* You might manage FK via triggers, other processes, or a second mapping

You won’t need this often early on — but it shows up when mapping legacy schemas.

---

### 2.6 `columnDefinition`

**What it does:**
Override the SQL definition of the FK column.

Example:

```java
@ManyToOne(optional = false)
@JoinColumn(
  name = "category_id",
  nullable = false,
  columnDefinition = "BIGINT NOT NULL"
)
private CategoryEntity category;
```

This is DB-specific and bypasses JPA’s type inference.

Use sparingly, mostly for:

* unusual types
* special defaults
* vendor-specific syntax

---

### 2.7 `table`

**What it does:**
Specifies the table that contains the FK column when using secondary tables (`@SecondaryTable`).

Example (rare):

```java
@Entity
@Table(name = "users")
@SecondaryTable(name = "user_details")
public class UserDetailsEntity {

  @Id
  private Long id;

  @OneToOne
  @JoinColumn(
    name = "user_id",
    table = "user_details"
  )
  private UserEntity user;
}
```

You probably won’t touch this for a long time. Don’t stress it.

---

### 2.8 `foreignKey`

**What it does:**
Controls the actual **FK constraint** definition.

```java
@ManyToOne(optional = false)
@JoinColumn(
  name = "category_id",
  nullable = false,
  foreignKey = @ForeignKey(name = "fk_products_category")
)
private CategoryEntity category;
```

Will generate:

```sql
ALTER TABLE products
ADD CONSTRAINT fk_products_category
FOREIGN KEY (category_id) REFERENCES categories(id);
```

Modes (`ConstraintMode`):

* `PROVIDER_DEFAULT` — let Hibernate decide
* `NO_CONSTRAINT` — no FK constraint generated
* `CONSTRAINT` — always generate FK

Many teams prefer to manage FKs via migrations instead and let Hibernate skip them, but this exists.

---

## 3) Typical Usages

---

### 3.1 Classic `@ManyToOne` (Product → Category)

```java
@Entity
@Table(name = "products")
public class ProductEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 200)
  private String name;

  @ManyToOne(fetch = FetchType.LAZY, optional = false)
  @JoinColumn(
    name = "category_id",
    nullable = false
  )
  private CategoryEntity category;
}
```

* FK: `products.category_id` → `categories.id`
* Relationship is required (cannot be null)

---

### 3.2 One-to-One via unique FK

```java
@Entity
@Table(name = "user_profiles")
public class UserProfileEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @OneToOne(fetch = FetchType.LAZY, optional = false)
  @JoinColumn(
    name = "user_id",
    nullable = false,
    unique = true
  )
  private UserEntity user;

  @Column(length = 500)
  private String bio;
}
```

This is often simpler than shared primary keys.

---

### 3.3 Self-reference via FK (`Category` parent → child)

```java
@Entity
@Table(name = "categories")
public class CategoryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 100)
  private String name;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "parent_id")
  private CategoryEntity parent;
}
```

FK: `categories.parent_id` → `categories.id`.

---

### 3.4 Side of `@JoinTable` (for ManyToMany)

Even though `@JoinTable` is a different annotation, it uses `@JoinColumn` inside:

```java
@ManyToMany
@JoinTable(
  name = "product_tags",
  joinColumns = @JoinColumn(name = "product_id"),
  inverseJoinColumns = @JoinColumn(name = "tag_id")
)
private Set<TagEntity> tags = new HashSet<>();
```

* `joinColumns` → FK to owning entity (`product_id`)
* `inverseJoinColumns` → FK to other side (`tag_id`)

---

## 4) Best Practices & Mental Model

### 4.1 Always name your join column

Bad (implicit):

```java
@JoinColumn
private CategoryEntity category;
```

Good (explicit):

```java
@JoinColumn(name = "category_id")
private CategoryEntity category;
```

Gives you stable schema & clear migrations.

---

### 4.2 Mirror `optional` and `nullable`

If relationship is required:

```java
@ManyToOne(optional = false)
@JoinColumn(name = "category_id", nullable = false)
```

If it’s optional:

```java
@ManyToOne(optional = true)
@JoinColumn(name = "parent_id", nullable = true)
```

Consistency makes your intent obvious.

---

### 4.3 Use `unique = true` for 1:1 via FK

```java
@OneToOne(optional = false)
@JoinColumn(
  name = "user_id",
  unique = true,
  nullable = false
)
```

---

### 4.4 Don’t overuse `columnDefinition`

Only when you have a real DB-specific need. Let JPA infer the type for normal FKs.

---

### 4.5 Let migrations own FK constraints (in more serious projects)

* Early on, letting JPA generate FK constraints is fine.
* In “grown-up” environments, Flyway/Liquibase migrations manage constraints.
* You still use `@JoinColumn` for ORM mapping, but schema is created by migrations.

---

### 4.6 Mental picture

For:

```java
@ManyToOne
@JoinColumn(name = "category_id")
private CategoryEntity category;
```

imagine:

* In **Java**: a field referencing another entity.
* In **DB**: a `category_id` column that points to `categories.id`.

`@JoinColumn` is the bridge between these two worlds.
