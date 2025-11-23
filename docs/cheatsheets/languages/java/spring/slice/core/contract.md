---
title: API Contracts
date: 2025-11-11
tags: 
    - spring
    - java
    - architecture
    - software-design
    - vertical-slice
summary: A practical guide to understanding and implementing API contracts in vertical slice architecture.
aliases:
    - API Contract Cheatsheet
---

# Understanding API Contracts for Vertical Slices

When building vertical slices in your backend, one concept that often comes up is the idea of an **API contract**.

It’s worth understanding **just enough** about “contract” to make your slices clean and predictable—but you don’t need to study it like a lawyer studying constitutional law. A contract in backend work is simply the shared promise your API makes to the outside world: *what the endpoint accepts, what it returns, and how it behaves when things go wrong.*

Once you grasp that, the whole idea becomes practical, not abstract.

Here’s the simplest useful way to look at it.

---

## What a contract really is

A contract is a **description of behavior at the boundary**. It says:

* “If you send me this shape of JSON…”
* “I will respond with this shape…”
* “And when something breaks, I will respond in this predictable way…”
* “And these status codes always mean the same thing…”

It is not fancy; it’s your controller’s “public language.”

Developers don’t come up with contracts by magic. They emerge because without them:

* endpoints become inconsistent,
* clients break silently,
* errors become random,
* versioning becomes impossible.

The contract is the thing that stops chaos.

---

## How deep do you need to go?

Deep enough to write a **precise example** of what an endpoint does before you code it. That’s it.

Everything else—OpenAPI, Swagger, schema evolution, versioning—comes later. Your vertical slice cares about a tiny piece:

**“What exactly happens when I call this endpoint with valid or invalid input?”**

If you can articulate that clearly, you are already doing contract-first API design.

---

## What the contract includes (in your beginner-friendly context)

Keep it to these essentials:

1. **The route** e.g., `POST /api/categories`
2. **The request JSON shape** `{"name": "Books"}`
3. **The success response**

    - Status code, headers, and body (if any).
    - For example: `201 Created` + `Location: /api/categories/{id}`.

4. **The error responses**

    - `409` when duplicate
    - `400` when validation fails  
      Both described using predictable `ProblemDetail`.

5. **Constraints/invariants**

    - name trimmed
    - name not blank
    - normalized uniqueness

That’s a contract. Nothing more mystical than that.

---

## Where does the contract begin?

At the API boundary—the controller.

**“What shape comes in?”**
**“What shape goes out?”**
**“How are errors expressed?”**

That’s the whole contract.

---

## Where does it end?

Once:

* the examples are written,
* the controller + DTOs implement them,
* the tests confirm they match the behavior.

At that moment, the contract is real, not theoretical.

---

## Should you study contracts beyond this?

Eventually yes, but only when you need it.

When you start dealing with:

* API versioning,
* schema evolution,
* backward compatibility,
* client libraries,
* event contracts,
* OpenAPI documentation,
* cross-team interfaces,

then understanding contracts at a deeper level becomes powerful.

But none of that is needed for your **first vertical slice**. You’ll just drown in theory before touching code.

---

## For your very next step

A contract for **CreateCategory** can be written like this (you can even keep it in your notebook):

```
POST /api/categories

Request:
{
  "name": "Books"
}

Success:
201 Created
Location: /api/categories/{id}

Duplicate:
409 Conflict
{
  "type": "/problems/duplicate-category",
  "title": "Duplicate category",
  "detail": "Category already exists",
  "instance": "/api/categories"
}

Validation error:
400 Bad Request
{
  "type": "/problems/validation-error",
  "errors": [
    { "field": "name", "message": "must not be blank" }
  ]
}
```

With that written down, you’ve defined the contract. You can now implement without guessing.

---

## The punchline

You do **not** need to go deeper than this right now. Understanding the contract at this level gives you everything you need to write clean slices.

Contracts evolve naturally when your project grows. Right now, you’re learning the rhythm: define → implement → test → deliver.

Once you feel that rhythm, deeper contract theory will start making sense instead of feeling abstract.
