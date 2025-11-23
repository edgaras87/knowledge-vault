---
title: Entity Dependencies
date: 2025-11-11
---



# Designing Entity Dependencies: Product & Category Example

When building an e-commerce system, a common question arises around how to model the relationship between products and categories. This cheat sheet explores best practices for defining this dependency in your database schema and JPA mappings.

## Should a product be allowed to exist without a category?

**It can, but usually shouldn’t.** You have three common models:

1. **Exactly one category (recommended for v1)**

    * DB: `products.category_id BIGINT NOT NULL` with a foreign key.
    * JPA: `@ManyToOne(optional = false)` + `@JoinColumn(nullable = false)`.
    * Pros: simpler queries, clear browsing/navigation, fewer edge-cases.
    * Cons: you need a category ready when creating a product.
    * Tip: create a default “Uncategorized” category at bootstrap if you want convenience while keeping `NOT NULL`.

2. **Zero-or-one category (nullable one-to-many)**

    * DB: `category_id` is **NULLABLE**.
    * JPA: `@ManyToOne(optional = true)`.
    * Pros: you can create products before taxonomy decisions.
    * Cons: more “unknown” states; every query & UI must handle `null`.
    * Use when your taxonomy is genuinely TBD in early prototyping.

3. **Many-to-many (products in multiple categories)**

    * DB: `product_categories(product_id, category_id)` join table; drop `category_id` from `products`.
    * JPA: `@ManyToMany`.
    * Pros: flexible navigation like “Shoes” and “Sale” simultaneously.
    * Cons: more joins, more complexity; pick this only when you really need it.
    * Hybrid pattern in mature systems: many-to-many **plus** a `primary_category_id` on `products` for canonical URLs and SEO.

## Practical starting advice

* **Start with (1) “exactly one”** or “exactly one with an 'Uncategorized' default.”
  It maximizes clarity and keeps your first slices lean.
* If later you decide products can belong to multiple categories, migrate to (3) with a safe path (below).

## Schema & mapping snippets

**Mandatory category**

```sql
-- categories
create table categories (
  id bigserial primary key,
  name varchar(100) not null,
  normalized_name varchar(100) not null unique
);

-- products
create table products (
  id bigserial primary key,
  name varchar(150) not null,
  normalized_name varchar(150) not null,
  price_cents integer not null,
  category_id bigint not null references categories(id),
  unique (category_id, normalized_name) -- names unique within a category
);
```

```java
// ProductEntity.java
@ManyToOne(optional = false, fetch = FetchType.LAZY)
@JoinColumn(name = "category_id", nullable = false)
private CategoryEntity category;
```

**Optional category**

```sql
alter table products add column category_id bigint null references categories(id);
-- (no NOT NULL)
```

```java
@ManyToOne(optional = true, fetch = FetchType.LAZY)
@JoinColumn(name = "category_id")
private CategoryEntity category; // may be null
```

**Many-to-many**

```sql
create table product_categories (
  product_id bigint not null references products(id),
  category_id bigint not null references categories(id),
  primary key (product_id, category_id)
);
```

```java
@ManyToMany
@JoinTable(name = "product_categories",
  joinColumns = @JoinColumn(name = "product_id"),
  inverseJoinColumns = @JoinColumn(name = "category_id"))
private Set<CategoryEntity> categories = new HashSet<>();
```

## Deletion rules (decide early)

* With mandatory FK: **restrict** category delete if products exist, or **require reassignment** first.
* With optional FK: you could `ON DELETE SET NULL`, but that creates “orphaned” products—often not what you want.
* Many-to-many: deleting a category should cascade **only** the join rows, not products.

## Zero-downtime evolution paths (expand → migrate → contract)

**A) Start without categories → later make it mandatory**

1. `ALTER TABLE products ADD COLUMN category_id BIGINT NULL;`
2. Backfill: create “Uncategorized” and set each product’s `category_id` to it.
3. Deploy code that writes `category_id` going forward.
4. After traffic confirms: `ALTER TABLE products ALTER COLUMN category_id SET NOT NULL;`
   Optionally add `UNIQUE (category_id, normalized_name)`.

**B) One category → later many-to-many**

1. Create `product_categories` and backfill from `products.category_id`.
2. Update code to use join table for reads/writes.
3. After confidence: drop `products.category_id`.

## Why start with Category first?

* It’s a tiny slice with a clear invariant (unique normalized name).
* It lets Product reference something real immediately.
* You’ll set up migrations, validation, ProblemDetail once, then reuse the pattern.

## Bottom line

* **Yes, a product *can* live without a category** (nullable FK), but you’ll pay in ambiguity and conditionals.
* For a beginner-friendly, clean path: **require a category** (or auto-assign “Uncategorized”). You get simpler queries, cleaner UI, and stronger integrity from day one.
* If your taxonomy needs to evolve, you have safe migrations to loosen the model later.

Take Categories today, wire the uniqueness + ProblemDetail, then add Products with a **NOT NULL** `category_id`. If/when real requirements push you to optional or many-to-many, you’ll migrate confidently.
