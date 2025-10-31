---
title: Correctness & Predictability 
date: 2025-10-31
tags:
    - spring
    - api-design
    - data-validation
    - best-practices
    - software-architecture
    - dto
summary: Your definitive map for building systems that never rot. Clean data in, truth enforced, invariants protected.
aliases:
    - Input Correctness and Predictability Cheatsheet
---

# API Input Contracts: L0 → L6 — Correctness and Predictability

### 🎯 Mission

Accept only **clean**, **valid**, and **truthful** data.

Pipeline:

```
Sanitize → Validate → Enforce domain truth → DB guarantees
```

---

### 🚪 Edge (API Boundary)

**Must happen first**

* Trim, NFC normalize (`Text.normalize`)
* Convert empty to null (except passwords)
* Reject invalid shapes early

**Tools**

* Global Jackson String sanitizer
* `@Valid` DTOs (`@NotBlank`, `@Size`, `@Email`, etc.)
* ProblemDetail errors

**Rule**

```
Sanitize FIRST → THEN @Valid → THEN map
```

---

### 🧠 Domain (Internal Truth)

Value Objects enforce invariants:

```
EmailAddress.of(raw) → normalize + validate
Slug.of(raw) → canonicalize + validate
```

Services accept **VOs, not raw strings**.

**Domain rule**

> If internal code wants to pass dirty input — it **must fail**.

---

### 🧱 Database (Final Wall)

Use DB to ensure nothing illegal survives:

* `NOT NULL`
* `UNIQUE`
* `lower(name)` unique indexes
* Foreign keys

DB rule:

> If app forgets a rule, DB still refuses garbage.

---

### 🔁 The Ladder (L0 → L6)

| Level | Name            | What you ensure                         |
| ----- | --------------- | --------------------------------------- |
| L1    | Hygiene         | Clean input (trim/NFC)                  |
| L2    | Validation      | Structure OK (`@Valid`)                 |
| L3    | Business Rules  | Logic correct (uniqueness, ranges)      |
| L4    | Tests           | It stays correct forever                |
| L5    | DB constraints  | Can't violate rules                     |
| L6    | Boundary Guards | MIME type, pagination caps, strict JSON |

**Execution order in real life:**
`Hygiene → Validation → Domain logic → DB`

---

### 🧠 Quick Placement Rules

| Concern                 | Layer             |
| ----------------------- | ----------------- |
| Trim / NFC              | JSON deserializer |
| `@NotBlank`, `@Size`    | DTO               |
| Unique / business truth | Service           |
| Permanent truth         | DB                |

---

### ⭐ Practical Mapping Table

| Field       | Clean               | Validate          | Domain?        | DB               |
| ----------- | ------------------- | ----------------- | -------------- | ---------------- |
| Name        | trim + NFC          | @NotBlank + @Size | optional VO    | varchar + length |
| Email       | trim + NFC          | @Email            | ✅ VO           | unique index     |
| Password    | **no trim**         | length only       | hash/salt only | N/A              |
| Slug        | lowercase + hyphens | @Size             | ✅ VO           | unique index     |
| Description | trim/NFC only       | —                 | string         | text             |

---

### 🧪 Tests to always write

* `"   John "` → `"John"`
* `"Café\u200D"` → `"Café"`
* Blank/whitespace → 400
* Duplicate canonical name → 409
* Domain VO rejects bad email even outside HTTP

---

### 🧵 One-sentence version

> **Clean at the door, prove truth inside, enforce in stone.**

---

Now your system remembers the philosophy **and** you get a tiny blade you can draw in 3 seconds when coding.

---

