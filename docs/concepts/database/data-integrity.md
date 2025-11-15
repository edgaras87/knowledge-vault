---
title: Data Integrity 
date: 2025-11-15
tags: 
  - database
  - data integrity 
  
summary: An in-depth explanation of data integrity in databases, its importance, types, and how databases enforce it to ensure accurate and reliable data.
aliases:
  - Data Integrity — Concept Explained
---


# Data Integrity — Concept Explained

Data integrity is the guarantee that **your data stays correct, consistent, and trustworthy** no matter how messy the world outside your database becomes.

It is the database's way of saying:

> “I will not let garbage enter.
> I will not let contradictions exist.
> I will defend the truth even if your application screws up.”

It is the backbone of reliable systems.

---

## 1. Why data integrity matters

Imagine an online store where:

* two products share the same slug
* an order references a user that doesn’t exist
* an invoice total doesn’t match line items
* emails are duplicated
* money is subtracted twice

This isn’t just “bad data.”
This is business chaos.

Without integrity, you lose trust in your system.
With integrity, you can build mountains on top of it.

---

## 2. The layers of data integrity

Data integrity is enforced by **multiple layers**, each with different strengths.

### A) Application Layer (your Java code)

* validates DTOs
* checks business rules
* prevents creating nonsense

Useful, but not final.

Why?
Because your application can fail: bugs, race conditions, concurrency issues.

### B) Database Layer (the last line of defense)

The database enforces rules that **cannot be bypassed**.

This is where real integrity lives.

---

## 3. Types of Data Integrity

### 3.1 Entity Integrity

Every row must be uniquely identifiable.

This is why we have:

* Primary keys (`id`)
* Auto-generated IDs
* No null PKs

If you can’t uniquely identify a row, you can’t update or join reliably.

---

### 3.2 Referential Integrity

Relationships between tables must make sense.

Example:

```
order.user_id   → must reference a user.id
product.category_id → must reference category.id
```

This is enforced with **foreign keys**.

Without this:

* You're able to create orders with no user
* Products referencing categories that no longer exist
* Orphans everywhere

The database uses cascading rules to protect or propagate changes.

---

### 3.3 Domain Integrity

Values must make sense in context.

Examples:

* price ≥ 0
* email is not null
* created_at must exist
* quantity must be integer
* enum must be from a known set
* slug must be unique

Enforced via:

* NOT NULL
* CHECK constraints
* UNIQUE constraints
* data types (INTEGER, VARCHAR, TIMESTAMP)
* enums (PostgreSQL has native ENUM types)

This ensures your data conforms to real-world rules.

---

### 3.4 Business Integrity (higher-level rules)

Uniqueness rules, invariants, assumptions that define your domain.

Examples:

* Each category slug is unique
* Customer cannot have two active carts
* Order total must equal sum of line items
* Stock cannot go below 0
* A user can have only one default address

These can be partially enforced by:

* UNIQUE constraints
* CHECK constraints
* foreign keys
* triggers (sometimes)
* or application-level logic + transactions

Business logic can be complex, so DB enforces what it can.

---

## 4. How databases enforce integrity

Databases have built-in mechanisms designed precisely for this purpose:

### ✦ Primary Key

Guarantees unique identity.

### ✦ Unique Index / Constraint

Prevents duplicates (slug, email, username).

### ✦ Foreign Key

Prevents invalid references.
Stops deleting referenced rows unless you cascade.

### ✦ NOT NULL

Prevents meaningless empty values.

### ✦ CHECK constraint

Prevents wrong or absurd values:

```sql
CHECK (price >= 0)
CHECK (quantity BETWEEN 1 AND 100)
```

### ✦ Types

Prevents storing “dog” in an INTEGER column.

### ✦ Transactions

Protects from partial writes:

* either all steps succeed
* or none do

This is integrity on a temporal axis.

---

## 5. Integrity vs Performance

Integrity has a cost:

* PKs → cheap
* FKs → restrict deletion / modifications
* UNIQUE → slows inserts
* CHECK → tiny overhead
* Transactions → coordination cost

But sacrificing integrity for speed is like removing seatbelts to make a car lighter.

Enterprise systems never do it.

---

## 6. Why we don’t rely only on Java for integrity

Because your code cannot guarantee correctness under:

* multiple instances of your service
* parallel requests
* race conditions
* deployment bugs
* application crashes
* network partition
* cron jobs
* manual database edits
* batch imports
* future developers making mistakes

The **database** is the final guardian.

---

## 7. Real-world example: category slug

### Without integrity

Two clients attempt to create category:

```
"Electronics" → slug: electronics
"Electronics" → slug: electronics (race condition)
```

Application checks:

* slug does not exist
* slug does not exist

Both pass.

Two inserts race.

### With `UNIQUE(slug)`

Database rejects the second insert with:

```
duplicate key value violates unique constraint "uk_categories_slug"
```

This is real integrity in action.

---

## 8. Mental model: “The database is the truth engine”

* Application tries to enforce rules
* Business tries to define rules
* Database guarantees rules cannot be broken

Integrity is the database refusing to lie.

---

## 9. Bottom line definition (one sentence)

> **Data integrity means your data always stays accurate, consistent, non-contradictory, and valid — no matter what your application code does.**

---

## 🧠 **Data Integrity — Q&A**

### ❓ What is data integrity?

It’s the guarantee that your data stays **correct, consistent, non-contradictory, and trustworthy**, no matter what your application code does.

---

### ❓ Who enforces data integrity?

The **database**.
Your app *attempts* it.
The DB *guarantees* it.

---

### ❓ Why isn’t the application layer enough?

Because it can’t prevent:

* race conditions
* concurrency conflicts
* multi-instance conflicts
* deployment bugs
* half-failed transactions

The database is the last line of defense.

---

### ❓ What happens if two requests race and try to insert duplicates?

Your app might fail to detect it.
The database **will not** miss it.

It rejects the second write with:

```
duplicate key value violates unique constraint
```

Correctness preserved.

---

### ❓ If the database enforces uniqueness, why does the app still check?

For **good user experience**:

* nice 409 ProblemDetail
* readable errors
* clear semantics
* faster failure (cheap existence check vs expensive SQL insert)

The DB check is correctness.
The app check is politeness.

---

### ❓ What types of integrity exist?

* **Entity integrity** → each row has a valid primary key
* **Referential integrity** → foreign keys reference valid rows
* **Domain integrity** → correct values (NOT NULL, CHECK, enums)
* **Business integrity** → domain rules (unique slug, one active cart)

---

### ❓ Why put integrity rules in the DB instead of the application?

Because only the database sees **all** writes from **all** sources at the same time.

Your app sees only its own request.

---

### ❓ What happens without data integrity?

Chaos:

* duplicate emails
* orphan orders
* invalid foreign keys
* wrong totals
* contradictory records
* nonsense data nobody can trust

