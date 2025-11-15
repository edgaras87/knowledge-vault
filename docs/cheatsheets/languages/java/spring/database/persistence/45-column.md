---
title: '@Column'
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@Column` annotation, detailing its attributes and usage for mapping entity fields to database columns.
aliases:
    - Jakarta Persistence — @Column Cheatsheet

---


# Jakarta Persistence — `@Column` Cheatsheet

Package: `jakarta.persistence.Column`

`@Column` configures how a **single Java field** maps to a **database column**:

* name
* NULL vs NOT NULL
* uniqueness (simple)
* type hints
* precision/scale
* whether JPA writes to it
* vendor-specific column definitions

It is the *core building block* of all entity→table mappings.

---

## 0) Contents

| Section                            | Description                     | Link                                                      |
| ---------------------------------- | ------------------------------- | --------------------------------------------------------- |
| **1) All Parameters**              | Full list of every attribute    | → [All Parameters](#1-all-parameters)                     |
| **2) Most Common Attributes**      | You use these daily             | → [Common Attributes](#2-most-common-attributes)          |
| **2.1 `nullable`**                 | NOT NULL constraint             | → [nullable](#21-nullable--not-null-constraint)           |
| **2.2 `length`**                   | VARCHAR length                  | → [length](#22-length--varchar-length)                    |
| **2.3 `name`**                     | Override column name            | → [name](#23-name--explicit-column-name)                  |
| **2.4 `unique`**                   | Single-column UNIQUE constraint | → [unique](#24-unique--simple-unique-constraint)          |
| **2.5 `precision` + `scale`**      | Decimal/money fields            | → [precision-scale](#25-precision--scale--numeric-fields) |
| **3) Less Common Attributes**      | Advanced but useful             | → [Less Common](#3-less-common-attributes)                |
| **3.1 `insertable`**               | Include column in INSERT        | → [insertable](#31-insertable--exclude-from-insert)       |
| **3.2 `updatable`**                | Include column in UPDATE        | → [updatable](#32-updatable--exclude-from-update)         |
| **3.3 `columnDefinition`**         | Raw SQL definition override     | → [columnDefinition](#33-columndefinition--raw-sql)       |
| **3.4 `table`**                    | Secondary tables                | → [table](#34-table--secondary-table)                     |
| **4) All Attributes in One Block** | Quick reference                 | → [All Attributes](#4-all-column-attributes-in-one-place) |
| **5) Practical Patterns**          | Ready-to-use setups             | → [Patterns](#5-practical-patterns)                       |
| **6) Best Practices**              | What to memorize                | → [Best Practices](#6-best-practices)                     |

---

## 1) All Parameters

```java
@Column(
    name = "",
    nullable = true,
    unique = false,
    insertable = true,
    updatable = true,
    columnDefinition = "",
    table = "",
    length = 255,
    precision = 0,
    scale = 0
)
```

---

## 2) Most Common Attributes

### 2.1 `nullable` — NOT NULL constraint

Controls whether DB allows NULL.

```java
@Column(nullable = false)
private String name;
```

DB:

```sql
name VARCHAR(255) NOT NULL
```

Use it for required fields. Not the same as `@NotNull`.

---

### 2.2 `length` — VARCHAR length

For `String` fields only.

```java
@Column(length = 100)
private String title;
```

DB:

```sql
title VARCHAR(100)
```

You’ll use this constantly.

---

### 2.3 `name` — explicit column name

Useful for snake_case DB columns.

```java
@Column(name = "created_at", nullable = false)
private Instant createdAt;
```

Without it, name depends on naming strategy.

---

### 2.4 `unique` — simple unique constraint

```java
@Column(unique = true, length = 100)
private String email;
```

DB:

```sql
email VARCHAR(100) UNIQUE
```

**Note:** For serious unique constraints (multiple columns / clearer naming), prefer:

```java
@Table(indexes = @Index(name = "...", columnList = "email", unique = true))
```

---

### 2.5 `precision` & `scale` — numeric fields

For `BigDecimal`.

```java
@Column(precision = 10, scale = 2)
private BigDecimal price;
```

Means:

* up to 10 digits total
* 2 after the decimal
* 8 before it

DB:

```sql
price DECIMAL(10, 2)
```

Perfect for money.

---

## 3) Less Common Attributes

### 3.1 `insertable` — exclude from INSERT

```java
@Column(insertable = false)
private Instant createdAt;
```

Useful when DB generates default values:

```sql
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
```

JPA will read but not write during insert.

---

### 3.2 `updatable` — exclude from UPDATE

```java
@Column(updatable = false)
private String createdBy;
```

Good for audit fields that must never change.

---

### 3.3 `columnDefinition` — raw SQL override

This bypasses JPA’s type inference:

```java
@Column(columnDefinition = "TIMESTAMP WITH TIME ZONE")
private Instant createdAt;
```

Or:

```java
@Column(columnDefinition = "BOOLEAN DEFAULT TRUE")
private Boolean active;
```

Use sparingly: vendor-specific, harder to migrate.

---

### 3.4 `table` — secondary tables

Used with `@SecondaryTable`.

```java
@Column(table = "user_details", name = "bio")
private String bio;
```

You won’t touch this early on.

---

## 4) All `@Column` Attributes in One Place

```java
@Column(
    name = "",             // explicit column name
    nullable = true,       // NULL allowed?
    unique = false,        // UNIQUE constraint?
    insertable = true,     // included in INSERT?
    updatable = true,      // included in UPDATE?
    columnDefinition = "", // raw SQL override
    table = "",            // secondary table name
    length = 255,          // VARCHAR length
    precision = 0,         // total digits (BigDecimal)
    scale = 0              // digits after decimal
)
```

**Sorted by usage frequency:**

* Daily:

  * `nullable`
  * `length`
  * `name`
  * `unique`
* Somewhat common:

  * `precision`, `scale`
* Advanced:

  * `insertable`, `updatable`
  * `columnDefinition`
  * `table`

---

## 5) Practical Patterns

### 5.1 Slug / email field

```java
@Column(nullable = false, length = 100, unique = true)
private String slug;
```

### 5.2 Money field

```java
@Column(nullable = false, precision = 12, scale = 2)
private BigDecimal price;
```

### 5.3 Audit columns

```java
@Column(name = "created_at", nullable = false, updatable = false)
private Instant createdAt;

@Column(name = "updated_at", nullable = false)
private Instant updatedAt;
```

### 5.4 DB default field

```java
@Column(
  insertable = false,
  updatable = false,
  columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP"
)
private Instant createdAt;
```

### 5.5 Optional text fields

```java
@Column(length = 500)
private String description;
```

---

## 6) Best Practices

### 1. Always control column lengths

Never rely on default VARCHAR(255) everywhere.

### 2. Use DB constraints for real integrity

`nullable = false` & unique constraints are your safety net.

### 3. Use `precision/scale` for money

Avoid floating-point values.

### 4. Avoid `columnDefinition` unless necessary

Vendor lock-in is a real risk.

### 5. Keep naming explicit

Use `@Column(name = "created_at")` for clarity.

### 6. Entity annotations describe schema

DTO annotations (validation) describe API contracts.
Do not mix them.





