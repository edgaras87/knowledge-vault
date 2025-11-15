---
title: Quick Starter Example
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: A quick starter example demonstrating common Jakarta Persistence (JPA) annotations through a simple domain model with entities, relationships, value objects, and enums.
aliases:
  - Jakarta Persistence — Quick Starter Example

---


# Jakarta Persistence — Quick Starter Example

Goal: **one small domain** using the most common JPA annotations:

* `@Entity`, `@Table`, `@Id`, `@GeneratedValue`
* `@Column`, `@Enumerated`, `@Embedded`, `@Embeddable`
* `@ManyToOne`, `@OneToMany`, `@JoinColumn`
* `@PrePersist` (lifecycle)

Domain:

* `Category` ←→ `Product` (many products in one category)
* `Product` has `Money` value object and `ProductStatus` enum

---

## 1. Folder structure (Java side)

```text
src/main/java/com/example/shop/infra/jpa/
  category/
    CategoryEntity.java
  product/
    ProductEntity.java
    Money.java
    ProductStatus.java
```

---

## 2. Category entity (basic mapping + index + lifecycle)

```java
// src/main/java/com/example/shop/infra/jpa/category/CategoryEntity.java
package com.example.shop.infra.jpa.category;

import jakarta.persistence.*;

import java.time.Instant;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(
    name = "categories",
    indexes = {
        @Index(
            name = "uk_categories_slug",
            columnList = "slug",
            unique = true
        )
    }
)
public class CategoryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;  // PRIMARY KEY

  @Column(nullable = false, length = 100)
  private String name;

  @Column(nullable = false, length = 100, unique = true)
  private String slug;  // business unique key

  @Column(name = "created_at", nullable = false, updatable = false)
  private Instant createdAt;

  // inverse side of relationship (Category → Products)
  @OneToMany(mappedBy = "category", fetch = FetchType.LAZY)
  private Set<ProductEntity> products = new HashSet<>();

  protected CategoryEntity() {
    // required by JPA
  }

  public CategoryEntity(String name, String slug) {
    this.name = name;
    this.slug = slug;
  }

  @PrePersist
  void onInsert() {
    this.createdAt = Instant.now();
  }

  // getters/setters (or Lombok)
}
```

---

## 3. Money as an embeddable value object

```java
// src/main/java/com/example/shop/infra/jpa/product/Money.java
package com.example.shop.infra.jpa.product;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;

import java.math.BigDecimal;

@Embeddable
public class Money {

  @Column(name = "price_amount", nullable = false, precision = 12, scale = 2)
  private BigDecimal amount;

  @Column(name = "price_currency", nullable = false, length = 3)
  private String currency;

  protected Money() {
  }

  public Money(BigDecimal amount, String currency) {
    this.amount = amount;
    this.currency = currency;
  }

  // getters
}
```

---

## 4. Product status enum

```java
// src/main/java/com/example/shop/infra/jpa/product/ProductStatus.java
package com.example.shop.infra.jpa.product;

public enum ProductStatus {
  DRAFT,
  ACTIVE,
  DISCONTINUED
}
```

---

## 5. Product entity (most annotations in one shot)

```java
// src/main/java/com/example/shop/infra/jpa/product/ProductEntity.java
package com.example.shop.infra.jpa.product;

import com.example.shop.infra.jpa.category.CategoryEntity;
import jakarta.persistence.*;

import java.time.Instant;

@Entity
@Table(
    name = "products",
    indexes = {
        @Index(name = "ix_products_slug", columnList = "slug"),
        @Index(name = "ix_products_category_id", columnList = "category_id")
    }
)
public class ProductEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 200)
  private String name;

  @Column(nullable = false, length = 150, unique = true)
  private String slug;

  // embedded value object → columns live in same table
  @Embedded
  private Money price;

  @Enumerated(EnumType.STRING)
  @Column(name = "status", nullable = false, length = 20)
  private ProductStatus status;

  @ManyToOne(fetch = FetchType.LAZY, optional = false)
  @JoinColumn(name = "category_id", nullable = false)
  private CategoryEntity category;

  @Column(name = "created_at", nullable = false, updatable = false)
  private Instant createdAt;

  @Column(name = "updated_at", nullable = false)
  private Instant updatedAt;

  protected ProductEntity() {
  }

  public ProductEntity(
      String name,
      String slug,
      Money price,
      ProductStatus status,
      CategoryEntity category
  ) {
    this.name = name;
    this.slug = slug;
    this.price = price;
    this.status = status;
    this.category = category;
  }

  @PrePersist
  void onInsert() {
    Instant now = Instant.now();
    this.createdAt = now;
    this.updatedAt = now;
  }

  @PreUpdate
  void onUpdate() {
    this.updatedAt = Instant.now();
  }

  // getters/setters
}
```

---

## 6. What you just used (quick legend)

From this one mini-example you covered:

**Entity & table**

* `@Entity` – marks class as JPA entity.
* `@Table(name = ..., indexes = ...)` – table name + indexes.

**Primary key & ID generation**

* `@Id` – primary key.
* `@GeneratedValue(strategy = GenerationType.IDENTITY)` – auto-increment PK.

**Columns & types**

* `@Column(nullable = false, length = X, unique = true, precision = 12, scale = 2, updatable = false, name = "...")` – shape + constraints for columns.
* `@Enumerated(EnumType.STRING)` – enum as text in DB.
* `@Embedded` + `@Embeddable` – value object flattened into same table.

**Relationships**

* `@ManyToOne(fetch = LAZY, optional = false)` – FK from Product → Category.
* `@JoinColumn(name = "category_id", nullable = false)` – actual FK column.
* `@OneToMany(mappedBy = "category", fetch = LAZY)` – inverse side Category → Products.

**Indexes**

* `@Index(name = "...", columnList = "...", unique = true/false)` – DB indexes & unique constraints via `@Table(indexes = {...})`.

**Lifecycle**

* `@PrePersist`, `@PreUpdate` – automatic timestamps.

