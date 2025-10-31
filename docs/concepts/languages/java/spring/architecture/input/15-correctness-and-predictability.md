---
title: Correctness & Predictability
date: 2025-10-30
tags: 
    - spring
    - api-design
    - data-validation
    - best-practices
    - software-architecture
    - dto
    
summary: Your definitive map for building APIs that never rot. Clean data in, truth enforced, invariants protected.
aliases:
    - API Input Correctness and Predictability
---

# API Input Contracts: L0 → L6 — Correctness and Predictability

Definitive map for building APIs that never rot.
Clean data in, truth enforced, invariants protected.

---

## 🧭 The Core Principle

> **Clean at the door (DTO)** → **Prove truth inside (Service)** → **Protect invariants in stone (DB)**

---

## 🧩 The Ladder (L0 → L6)

| Level                           | Focus                                                 | Layer               | Description                                                         |
|---------------------------------| ----------------------------------------------------- | ------------------- | ------------------------------------------------------------------- |
| **L0 – Baseline**               | It compiles, happy path works.                        | —                   | Minimum viable code runs.                                           |
| **L1 – Hygiene**                | Data normalization                                    | DTO                 | Trim, NFC normalize, use ISO-8601, `BigDecimal`, `E.164`.           |
| **L2 – Validation**             | Bean Validation (`@NotBlank`, `@Size`, `@Past`, etc.) | DTO                 | Checks structure and basic constraints.                             |             
| **L3 – Business Rules**         | Logical and domain truth                              | Service             | Cross-field checks, uniqueness, date logic.                         |
| **L4 – Tests**                  | Regression and boundary verification                  | HTTP Tests          | MockMvc or integration tests for 400/201 correctness.               |
| **L5 – Persistence Invariants** | Database guarantees                                   | Entity / DB         | `nullable=false`, unique indexes, foreign keys, constraints.        |
| **L6 – Boundary Guards**        | API layer defense                                     | Controller / Config | Content-type limits, pagination caps, strict JSON, `ProblemDetail`. |

**Shortcut rule:**
L0–L4 are your **daily reps**.
L5 and L6 are **insurance** against future developer errors.

---

## 🧠 Layer Responsibilities

| Layer                   | Responsibility    | Examples                                   |
| ----------------------- | ----------------- | ------------------------------------------ |
| **DTO**                 | Shape + Hygiene   | Trim, normalize, validate with annotations |
| **Service**             | Business Truth    | “start ≤ end”, uniqueness, domain logic    |
| **Entity / DB**         | Hard Guarantees   | `NOT NULL`, unique index, foreign key      |
| **Controller / Config** | Guardrails        | Whitelist params, JSON strict mode         |
| **Tests**               | Regression Alarms | 400/201 consistency checks                 |

---

## 🔬 Why This Works

* **L1 + L2:** Stop garbage at the door.
* **L3:** Encodes product truth in the right layer.
* **L4:** Makes regressions loud.
* **L5:** Guarantees DB won’t accept what app rejects.
* **L6:** Blocks abuse (mass pagination, wrong MIME types, etc.).

---

## 🪶 The Mental Tattoo

* **DTO:** “Is this data clean and understandable?”
* **Service:** “Is this data *true in our world*?”
* **DB:** “Even if future devs are sloppy, this rule cannot break.”

Examples:

* `"   John "` → trim in DTO.
* `birthDate` → parse in DTO, **age < 120** in Service.

DTOs know **shape**, not **meaning**.
Services decide **truth**.
Database enforces **permanence**.

---

## 🧩 Rule Taxonomy

| Kind of Rule             | Example                        | Layer           |
| ------------------------ | ------------------------------ | --------------- |
| **Shape**                | Must be string/date/number     | Framework + DTO |
| **Cleanliness**          | Trim, NFC, no blanks           | DTO             |
| **Basic Semantic Hint**  | `@Past`                        | DTO             |
| **Logical Truth**        | Age < 120, booking end > start | Service         |
| **Cross-Field**          | password != username           | Service         |
| **Cross-Entity**         | email unique                   | Service + DB    |
| **Absolute Enforcement** | NOT NULL, UNIQUE, FK           | DB              |

---

## 🧱 Default Placement Guide

| Type                    | Layer   |
| ----------------------- | ------- |
| **DTO**                 | L1 + L2 |
| **Service**             | L3      |
| **Entity / DB**         | L5      |
| **Controller / Config** | L6      |
| **Tests**               | L4      |

---

## 🧮 Quick Archetypes

| Field     | Boundary Rule (DTO)        | Domain Rule (Service)  |
| --------- | -------------------------- | ---------------------- |
| **name**  | trim, `@NotBlank`, `@Size` | —                      |
| **date**  | valid ISO, `@Past`         | age ≤ 120, not in past |
| **email** | `@Email`, lowercase domain | unique, verified       |
| **money** | decimal shape              | ≥ 0 balance            |
| **id**    | UUID format                | unique index           |
| **enum**  | must match value           | invalid → 400          |

---

| Field Type | Recommended Practice (DTO level) |
|-------------|----------------------------------|
| **String (ordinary)** | trim + NFC, `@NotBlank/@Size`, optional `@Pattern` |
| **Password / Secret** | no trim, only length validation (`@RawInput` marker) |
| **Email** | lowercase domain only, `@Email`, external verification |
| **Phone** | parse to E.164, `@Pattern` or `libphonenumber` |
| **Money** | `BigDecimal`, scale 2, `@DecimalMin("0.00")`, `@Digits(...,2)` |
| **Date / Time** | ISO-8601, `@Past/@Future`, store UTC |
| **IDs** | UUID format, validate pattern, handle 404/409 existence |
| **Enums** | strict enum mapping, reject unknown → 400 |

*(These define L1 + L2 rules — cleanliness and structure before business logic.)*

---

## 🧰 Example Template (for your vault)

Use this to define each field explicitly.

```
### Field: name (String)

Contract
- Required? yes
- Format: human name, max 64 chars

Hygiene (DTO)
- Trim → blanks to null
- Unicode NFC
- Preserve case

Validation (DTO)
- @NotBlank, @Size(max=64)
- Optional: @Pattern for allowed chars

Business Rules (Service)
- Uniqueness? no
- Additional? none

Persistence (Entity/DB)
- @Column(nullable=false, length=64)
- Index/unique? no

Boundary Guards
- none

Tests
- POST trims + normalizes → 201 body “Café”
- POST blank → 400 validation ProblemDetail

Gotchas
- Zero-width spaces; double spaces allowed
```

You can clone this template for other fields (`email`, `birthDate`, `price`, etc.).
Over time you’ll build your **Field Catalog** — your personal API grammar book.

---

## 🧩 Quick Summary: The Three Pillars

| Phase       | Question                     | Example                 |
| ----------- | ---------------------------- | ----------------------- |
| **DTO**     | “Is it clean and valid?”     | `"   John "` → `"John"` |
| **Service** | “Is it logical and allowed?” | age < 120               |
| **DB**      | “Can this ever break later?” | `unique(email)`         |

This ladder (L0–L6) gives *predictability, correctness,* and *defense in depth* — the difference between “it works” and “it never fails in production.”

---















