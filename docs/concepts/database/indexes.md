---
title: Database Index 
date: 2025-11-15
tags: 
  - database
  - indexing
  - performance 
  
summary: An in-depth explanation of database indexes, their purpose, structure, and impact on query performance.
aliases:
  - Database Index — Concept Explained
---

# Database Index — Concept Explained

### What an index *is*

An index is a **data structure** the database keeps next to your table so it can **find rows faster**.

When your table is small, the database can simply read every row (“table scan”).
When your table grows to millions of rows, that becomes painfully slow.

An index solves that.

---

## The textbook analogy

Imagine a 500-page book:

* Without an index → To find “cosmic microwave background” you read every page.
* With an index → Flip to the back, find the term instantly, jump to the exact page.

Database does the same thing.

---

## Why indexes matter

Without index:

```
SELECT * FROM users WHERE email = 'test@example.com';
```

The database must read **every row** in `users`.

With index on `email`:

* DB jumps directly to the matching positions in the index tree
* Retrieves rows in logarithmic time, not linear

**Performance difference is enormous.**
Millions of rows? Still fast.

---

## What an index really looks like (conceptually)

Most relational databases use **B-tree indexes**.

Picture a sorted tree:

```
             [a@, m@, s@]
            /     |     \
         ...     ...     ...
```

* Stored separately from the table
* Always sorted by the indexed column(s)
* Allows binary search instead of full scan

Some databases use alternate structures for special cases:

* Hash indexes (exact equality only)
* GIN/GiST (full text search)
* R-trees (geospatial)

But for normal filtering (email, slug, ID, timestamps), the mental model is **“ordered tree sorted by my column.”**

---

## How queries behave with vs without index

### WITHOUT index

Query:

```sql
SELECT * FROM products WHERE price < 100;
```

DB does a **full table scan**:

```
look at row1
look at row2
look at row3
...
```

### WITH index on `price`

DB navigates the **sorted index**, finds all entries under 100 directly, and fetches only those rows.

---

## Index vs Table: how they relate

Think of the table as the “full story” — all columns, all data.

The index is a **shortcut book** that contains only:

* The indexed column(s)
* A pointer to the real row in the table

For example, an index on `(email)` stores:

| email                                           | pointer   |
| ----------------------------------------------- | --------- |
| [adam@example.com](mailto:adam@example.com)     | → row 57  |
| [brenda@example.com](mailto:brenda@example.com) | → row 102 |
| [zorro@example.com](mailto:zorro@example.com)   | → row 8   |

Sorted by email.

---

## Cost of an index

Indexes speed up **reads**, but slow down **writes**.

Every INSERT / UPDATE / DELETE must also maintain the index.

For example:

```sql
INSERT INTO users (email) VALUES ('bob@example.com')
```

DB must:

1. Insert the row
2. Insert `'bob@example.com'` into the email index
3. Rebalance the tree

So adding *too many* indexes harms performance.
Good indexing is a balance.

---

## When indexes help (rules of thumb)

Indexes help when:

* You filter a column often:

  * `WHERE email = ?`
  * `WHERE slug = ?`
  * `WHERE created_at > ?`
* You sort often:

  * `ORDER BY created_at DESC`
* You join often:

  * `JOIN orders o ON o.user_id = u.id`
* You enforce uniqueness:

  * email unique
  * username unique
  * category slug unique

Indexes don’t help when:

* You filter low-selectivity data:

  * `WHERE gender = 'M'` (half the table → DB may still scan)
  * `WHERE is_active = true`
* Table is tiny (100 rows → scan is cheap)
* Column is never used in lookup

---

## Types of indexes (concept-level)

**Single-column index**
Sorted by one column.

**Composite (multi-column) index**
Sorted by `(column1, column2)`
Very powerful for “find by two properties.”

**Unique index**
Same as above, but enforces “no duplicates.”

**Clustered index**
(Used in SQL Server)
The physical row order follows the index.

**Non-clustered index**
Most common; normal index stored separately.

---

## What uniqueness actually protects

A `UNIQUE` index makes the database your bodyguard:

```sql
INSERT INTO users (email) VALUES ('a@b.com');
INSERT INTO users (email) VALUES ('a@b.com'); -- rejected
```

Even if your application has a bug, the DB protects your data.

---

## Why backend engineers must understand indexing

Because performance and correctness hinge on it.

Your API might look clean and elegant, but without proper indexing your application will choke on:

* filtering
* pagination
* joins
* existence checks
* uniqueness constraints
* foreign key navigation

Indexes are the quiet engine that keeps the highway moving.

---

## The bottom-line mental model

A **database index** is:

* A sorted structure
* Kept separately from the table
* Allowing fast lookup, filtering, and ordering
* At the cost of extra storage and slower writes

It’s one of the most fundamental concepts in all of backend engineering.

---

## Database Index — Q&A Cheatsheet

---

### ❓ What is a database index?

A sorted data structure (usually a B-tree) the DB keeps separately from the table to make lookups fast.

---

### ❓ Why does the database need indexes?

Without them, every `SELECT` requires scanning *all rows*.
With indexes, the database can jump directly to matching rows.

---

### ❓ Do indexes store full data?

No.
They store:

* indexed column(s)
* pointer to the full row in the table

The table remains the “real data”.

---

### ❓ Does every column need an index?

No.
Only index columns you frequently:

* filter by (`WHERE`)
* join on
* sort by (`ORDER BY`)
* enforce uniqueness on

Everything else remains unindexed.

---

### ❓ Are indexes stored inside the table?

No.
A table is one structure; indexes are separate structures maintained alongside it.

---

### ❓ What’s the downside of too many indexes?

Writes slow down.

Every time you:

* insert
* update
* delete

the DB must **update every index** that column participates in.

---

### ❓ Will adding an index always make queries faster?

No.

If a query returns a large percentage of the table (30–50%+),
the DB may prefer a full scan — even if the column is indexed.

---

### ❓ What is index selectivity?

The measure of how many unique values a column has.

Highly selective → many distinct values → good index
Low selective → few values → index often useless

---

### ❓ Should I index boolean fields?

Almost never.

`TRUE/FALSE` offers almost no filtering power.

---

### ❓ Should I index gender (male/female, 50/50)?

Almost always no.
The index splits rows into two large halves — not helpful.

---

### ❓ Should I index categories table?

Yes, but **for correctness**, not speed.

Example: category `slug` must be unique → unique index.

Even if table is tiny, uniqueness constraints belong in the database.

---

### ❓ Should I index the foreign key of a product table?

Yes.

Filtering products by category is extremely common:

```sql
WHERE category_id = ?
```

A foreign-key column almost always deserves an index.

---

### ❓ What is a composite index?

An index on multiple columns:

```sql
(category_id, status)
```

Useful when queries filter by **both** columns together.

---

### ❓ Does column order in a composite index matter?

Yes — critically.

Index `(a, b)` helps:

```
WHERE a = ? AND b = ?
WHERE a = ?
```

But **not**:

```
WHERE b = ?
```

Left-most rule.

---

### ❓ Should I make composite indexes with booleans?

Only if the *first* column is selective.

Index `(is_active, user_id)` → worthless
Index `(user_id, is_active)` → may be useful

---

### ❓ What is a unique index?

An index that enforces “no duplicates”.

Used for:

* slug
* email
* username
* order number
* product SKU

---

### ❓ Is `UNIQUE` only for performance?

No.

Its primary purpose is **data integrity**.
Performance boost is secondary.

---

### ❓ Why not just enforce uniqueness in Java?

Because Java can race.
The database is the final authority on data consistency.

---

### ❓ If a table is tiny, should I avoid indexes?

For performance → yes, not needed.
For correctness → add when required (unique constraints).

---

### ❓ Does an index speed up INSERTs?

No — it slows them down.

But usually the speed hit is tiny unless you have many indexes or huge write volume.

---

### ❓ Do indexes help with `ORDER BY`?

Yes — if the ordering matches the index.

`ORDER BY created_at DESC` + index on `(created_at)` → perfect.

---

### ❓ What if I don’t know what queries will look like yet?

Index the obvious things:

* primary keys
* foreign keys
* unique business identifiers (email, slug, sku)
* timestamps used for sorting

Add more later after observing query patterns.

---

### ❓ Can I remove indexes?

Yes.
Removing unnecessary indexes can dramatically speed up write-heavy systems.

---

### ❓ How do I know if an index is being used?

Use:

* PostgreSQL → `EXPLAIN` / `EXPLAIN ANALYZE`
* MySQL → `EXPLAIN`
* SQL Server → execution plan

They show whether the DB is using an index or scanning the table.

---

### ❓ What’s the “sweet spot” for number of indexes?

Enough to support your most common queries,
but not so many that your writes slow down.

Typical backend tables end up with:

* PK index
* FK indexes
* unique indexes for identifiers
* sometimes 1–2 more for lookup patterns
* and rarely more than 5–6 total

---

### ❓ What’s the biggest indexing mistake beginners make?

Two things:

1. Creating indexes without understanding selectivity
2. Failing to index foreign keys
3. (Bonus) Believing indexes are a magic speed hack — they’re not

---

### ❓ What’s the fastest way to think about indexes?

This one sentence:

> Indexes help WHEN they reduce the search space.
> If they don’t reduce much, they don’t help.

