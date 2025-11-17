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


## Field Types Cheatsheet

---

### 📌 Field Cheatsheet Template (all layers)

```
## <Type>: <fieldName>

Sanitize (DTO)
- <trim/NFC/case policy/parse strictly>

Validate (DTO)
- <Bean Validation annotations>
- <notes>

Service Rules (L3)
- <cross-field / domain truth>

Persistence (L5, Entity/DB)
- @Column(...) / type / length
- Index/Unique/FK

Boundary Guards (L6)
- <controller/config guardrails>

Tests (L4)
- "<input>" → <result/code>
- "<input>" → <result/code>
- Edge: <case>
```

---

### String: name

```
Sanitize (DTO)
- Trim → blank → null
- Unicode NFC
- Preserve case (do not auto-capitalize)

Validate (DTO)
- @NotBlank
- @Size(max = 64)
- Optional: @Pattern(...) if domain bans digits/symbols

Service Rules (L3)
- Usually none; never “auto-unique” by name

Persistence (L5, Entity/DB)
- VARCHAR(64) NOT NULL
- No unique index by default

Boundary Guards (L6)
- None

Tests (L4)
- "  John  " → cleaned "John" → 201
- "" / "   " → 400
- "Café" → 201
- 65 chars → 400
```

---

### Date: birthDate (LocalDate)

```
Sanitize (DTO)
- Strict ISO-8601 "YYYY-MM-DD"
- Reject ambiguous formats; no timezone for LocalDate

Validate (DTO)
- @Past
- @NotNull (if required)

Service Rules (L3)
- Age ≤ 120 (domain truth)
- If used for access: must be ≥ 18, etc.

Persistence (L5, Entity/DB)
- DATE NOT NULL
- No index unless queried often

Boundary Guards (L6)
- None

Tests (L4)
- "2000-01-01" → 201
- "01-01-2000" → 400 (format)
- "2099-01-01" → 400 (@Past)
```

---

### String: email

```
Sanitize (DTO)
- Trim + NFC
- Lowercase DOMAIN only (preserve local part)

Validate (DTO)
- @Email
- @Size(max = 128)
- Optional: @NotBlank

Service Rules (L3)
- Must be unique per user → check existence before insert
- Optional: deliverability/verification workflow

Persistence (L5, Entity/DB)
- VARCHAR(128) NOT NULL
- UNIQUE INDEX (email)

Boundary Guards (L6)
- Fail on unknown JSON props to avoid silent “email2”

Tests (L4)
- "  A@Example.COM  " → "A@example.com" → 201
- "invalid@" → 400
- Duplicate → 409 (or 400) with ProblemDetail
```

---

### Money: amount (BigDecimal)

```
Sanitize (DTO)
- Parse strictly to BigDecimal
- Normalize scale = 2; reject scientific notation

Validate (DTO)
- @DecimalMin("0.00")
- @Digits(integer=12, fraction=2)
- @NotNull if required

Service Rules (L3)
- Business invariants: balance ≥ 0, totals match, currency rules

Persistence (L5, Entity/DB)
- DECIMAL(14,2) NOT NULL
- Consider CHECK (amount >= 0)

Boundary Guards (L6)
- Reject text/float content-types; JSON number only

Tests (L4)
- "10.00" → 201
- "-1.00" → 400
- "12.345" → 400
```

---

### Phone: msisdn (E.164)

```
Sanitize (DTO)
- Trim + NFC; strip spaces/dashes/paren
- Normalize with libphonenumber to E.164 (+3706...)

Validate (DTO)
- @Pattern for E.164 (^\+?[1-9]\d{7,14}$)
- @Size(max=16)

Service Rules (L3)
- Uniqueness per account (if required)
- Country/plan eligibility checks

Persistence (L5, Entity/DB)
- VARCHAR(16) NOT NULL
- UNIQUE INDEX if used as contact key

Boundary Guards (L6)
- None

Tests (L4)
- " (370) 612 34567 " → "+37061234567" → 201
- "123" → 400
- Duplicate → 409
```

---

### ID: id (UUID)

```
Sanitize (DTO)
- Accept only canonical 36-char UUID string; no curly/braced forms

Validate (DTO)
- @Pattern for v4 if you enforce version
- @NotBlank

Service Rules (L3)
- Existence check: 404 if not found
- Authorization check: id must belong to requester’s scope

Persistence (L5, Entity/DB)
- UUID/BINARY(16) primary key
- FK for references with ON DELETE policy

Boundary Guards (L6)
- Fail on unknown fields; path-var vs body-id conflict → 400

Tests (L4)
- Non-UUID → 400
- Valid UUID not found → 404
- Path id ≠ body id → 400
```

---

### Enum: status

```
Sanitize (DTO)
- Map strictly to enum; no lenient/coercive mapping

Validate (DTO)
- @NotNull
- Use Jackson’s READ_UNKNOWN_ENUM_VALUES_AS_NULL=false

Service Rules (L3)
- State machine transitions (e.g., DRAFT → PUBLISHED allowed, not reverse)

Persistence (L5, Entity/DB)
- VARCHAR(24) NOT NULL (or SMALLINT with mapping table)
- Index if queried by status

Boundary Guards (L6)
- None

Tests (L4)
- Unknown value → 400
- Illegal transition → 409/422 with ProblemDetail
```

---

### Pagination: page/size (query params)

```
Sanitize (DTO)
- Parse ints; default page=0, size=20

Validate (DTO)
- @Min(0) for page
- @Min(1) @Max(100) for size

Service Rules (L3)
- Enforce max server-side regardless of client input

Persistence (L5, Entity/DB)
- (N/A)

Boundary Guards (L6)
- Cap size in controller; reject negative page
- Set sensible default sort

Tests (L4)
- size=100 → OK
- size=1000 → coerced to 100 or 400 (pick policy)
- page=-1 → 400
```

---

### File upload: photo (multipart)

```
Sanitize (DTO)
- Trust Content-Type? No → sniff first bytes (magic numbers)

Validate (DTO)
- Allowlist: image/jpeg, image/png
- Max size (e.g., 5 MB)

Service Rules (L3)
- Virus scan / image re-encode pipeline
- Derive width/height limits

Persistence (L5, Entity/DB)
- Store in object storage; DB keeps metadata
- DB constraints: NOT NULL filename, size, checksum

Boundary Guards (L6)
- Limit multipart count; disable unknown parts
- Set max request size in server config

Tests (L4)
- JPEG under limit → 201
- Wrong MIME / oversized → 400
- Corrupt file signature → 400
```

---

### Controller/Config knobs (drop-in once, reuse everywhere)

* **DTO validation on**: `@Valid @RequestBody`, `@Validated` on controller.
* **Strict JSON**: `FAIL_ON_UNKNOWN_PROPERTIES=true`, `FAIL_ON_NULL_FOR_PRIMITIVES=true`, `FAIL_ON_NUMBERS_FOR_ENUMS=true`.
* **ProblemDetail**: unify 400/404/409 with `@RestControllerAdvice`.
* **UTF-8 + NFC**: normalize input strings once (custom Jackson deserializer).

