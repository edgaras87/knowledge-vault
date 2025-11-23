---
title: Vertical Slice Concept
date: 2025-11-11
---


# What a vertical slice *is*

A **single, user-visible capability** that runs straight through your stack: 

HTTP contract → web edge (DTOs/validation/errors) → app/use-case → domain/rules → persistence/migration → tests → a touch of observability. 

It’s shippable on its own.

- **Starts:** when you define the contract and acceptance criteria.
- **Ends:** when code, migration, tests, and basic logs/metrics for *that capability* are done and the endpoint works.

# Is “Create Category” a vertical slice?

Yes. **Create Category** (POST `/categories`) is one vertical slice.  
“Update Category,” “Delete Category,” “Get Category,” “List Categories” are **separate** slices. You can group tiny reads together if you must, but treating each action as its own slice keeps scope tight and progress visible.

# What you should know/notate before you start (5 bullets max)

Write this into a short note; 5 minutes, not 50.

1. **Story**: “As an admin, I can create a category with a unique name.”
2. **Contract**: Request, Response, Status codes, Error cases (examples).
3. **Invariants**: trim, non-blank, unique on normalized name.
4. **Data change**: table + unique index you’ll add with Flyway.
5. **Done criteria**: 201 + `Location` works, 409 ProblemDetail on duplicate, tests green (unit/web/jpa/integration), one useful log/metric.

# The steps (run these every slice)

1. **Contract first** (examples):

    * POST `/api/categories` → `201 Created` + `Location: /api/categories/{id}`
    * Errors: `409` with `type:/problems/duplicate-category`
   
2. **Migration**: Flyway `V001__categories.sql` with `unique(normalized_name)`.
3. **Domain**: model rules (trim/normalize/validate).
4. **Repo + use-case**: check uniqueness, save, return id.
5. **Web edge**: request/response DTOs, validation, controller; ProblemDetail mapping (global advice).
6. **Tests**:

    * domain unit test (rules),
    * `@DataJpaTest` (unique index),
    * `@WebMvcTest` (201 + Location; 409 ProblemDetail shape),
    * one integration (happy path with Testcontainers).

7. **Observability**: log `category.created` (id, normalizedName); optional counter.
8. **Refactor pass (max 2 changes)**; park the rest in a tiny “research-debt” note.

# How to “define” a slice (copy-paste template)

```
# Slice: Create Category

## Story
As an admin, I create a category with a unique name.

## Contract
POST /api/categories
Request:
{ "name": "Books" }
Response:
201 Created
Location: /api/categories/{id}
Errors:
409 Conflict (type: /problems/duplicate-category, detail: "already exists")

## Invariants
- name trimmed, not blank
- normalized_name = lower(name)
- unique(normalized_name)

## Data
- Table: categories(id, name, normalized_name, created_at)
- Index: unique(normalized_name)

## Done
[ ] 201 + Location works
[ ] 409 ProblemDetail on duplicate
[ ] Tests: domain / DataJpa / WebMvc / integration
[ ] Log event category.created
```

# CRUD: one slice or many?

* **Create, Update, Delete, Get, List** → treat as **separate slices**.
* You *may* bundle **Get + List** if trivial and you want to move faster, but keep **Create** (and any mutating action) separate; they carry business rules and error handling you want to test cleanly.

# Quick sanity checks (so you don’t drift)

* Does this slice deliver a visible outcome? If not, it’s plumbing—re-scope.
* Can you delete the code for this slice without breaking others? Good; it’s vertical.
* Did you add/alter schema with Flyway? If not, you’re skipping data ownership.
* Do you have at least 3 fast tests + 1 integration? If not, you’ll fear refactors later.

# Apply it now

Start **Create Category** exactly as above. Don’t add security yet. When it’s green, move to **List Categories** as a second slice (with pagination). That rhythm builds the muscle you want—and you’ll *feel* where the design patterns fit, instead of hunting for a “perfect path.”
