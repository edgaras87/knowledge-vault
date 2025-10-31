---
title: Sanitation & Validation
date: 2025-10-31
tags: 
    - spring
    - api-design
    - data-validation
    - best-practices
    - software-architecture
    - dto
summary: Your definitive playbook for input sanitation and validation in Java Spring applications. Ensure clean data entry and enforce invariants across your system.
aliases:
    - Input Sanitation and Validation in Java Spring
---

# Input Sanitation & Validation — The Playbook

## Why this matters (the “physics” of text)

Real systems ingest messy text: trailing spaces, zero-width junk, weird Unicode forms, emoji, look-alikes. If you only clean at the **REST endpoint**, dirt can still enter from other doors: CLI scripts, batch jobs, Kafka consumers, test fixtures, even JPA reads/writes. That leads to:

* Broken uniqueness checks (“Books ” vs “Books”).
* Inconsistent comparisons (NFD vs NFC).
* User-visible glitches (“invisible space” bugs).
* Data you can’t reliably query.

So you need **two guardrails** with **one source of truth**.

---

## Two guardrails, one truth

### Guardrail 1 — Endpoint layer (UX + consistency at the edge)

Purpose: make incoming JSON sane and give clients friendly errors.

* **Normalize on deserialization**: a global Jackson `@JsonComponent` that runs `Text.normalize(String)` on every `@RequestBody` `String`. Add `@Raw` to skip sensitive fields (passwords/tokens).
* **Validate DTOs**: `@Valid` + `@NotBlank`, `@Size`, etc. Validation now runs on the cleaned text, so messages and limits are correct.
* **Map to domain**: use MapStruct (optional) to avoid boilerplate.

Outcome: clients get predictable 400s; your services see already-clean strings.

### Guardrail 2 — Internal/domain layer (ironclad invariants)

Purpose: protect **all** entry points (REST, CLI, Kafka, tests).

* Promote “hot” string concepts to **Value Objects (VOs)**: `EmailAddress`, `Slug`, maybe `NormalizedName`. Their `.of(raw)` does the same `Text.normalize(...)` and enforces rules.
* Make services accept **typed commands/VOs**, not raw strings. If a caller tries to pass junk, the VO constructor explodes before data enters the core.
* Optional last mile: `@PrePersist/@PreUpdate` or JPA `AttributeConverter` for VOs to keep DB round-trips canonical.

Outcome: even non-HTTP code can’t inject dirt.

**One source of truth:** `Text.normalize(...)` is used by both the endpoint (via Jackson) and the VO constructors. Change it once → the whole app adapts.

---

## What’s “good enough” for practice vs production?

### For practice / portfolio (ship fast, don’t drown)

* Use DTOs with Bean Validation at endpoints.
* Add a tiny sanitizer utility `Text.normalize(String)` and call it **in the service** for key fields you compare/query (e.g., `name` before `existsByName`).
* Create a DB unique index on the canonical form (e.g., `lower(name)`).

This is perfectly fine while you learn and build features.

### For production / long-lived systems (go full)

* **Endpoint:** global Jackson String normalizer (+ `@Raw` escapes), DTO validation.
* **Domain:** VOs for invariant concepts; services accept VOs (or commands composed of VOs).
* **Mapping:** MapStruct type-based converters to route `String → VO` everywhere without repetition.
* **Persistence:** DB unique indexes (e.g., `lower(name)`), matching column lengths.
* **Optional safety nets:** `@PrePersist/@PreUpdate` per entity or `AttributeConverter` per VO.

---

## What NOT to do (pain generators)

* Don’t rely **only** on `@PrePersist/@PreUpdate`: too late for validation and equality/uniqueness checks.
* Don’t sprinkle `normalize()` in **every constructor**: easy to miss paths, bad errors, duplication.
* Don’t create a VO **per entity field** (`CategoryEmail`, `UserEmail`…) — create a VO **per concept** (`EmailAddress`) and reuse it.

---

## Minimal code you can reuse

**1) Normalizer (one truth):**

```java
public final class Text {
  private Text() {}
  public static String normalize(String s) {
    if (s == null) return null;
    String t = s.trim()
      .replace("\u200B","").replace("\u200C","").replace("\u200D","");
    return java.text.Normalizer.normalize(t, java.text.Normalizer.Form.NFC);
  }
}
```

**2) Endpoint guard (global JSON normalizer + opt-out):**

* Jackson `@JsonComponent` String deserializer → runs `Text.normalize` for all `@RequestBody` strings.
* `@Raw` annotation to skip sensitive fields (passwords, signatures, base64).

**3) Domain VOs (pick only the concepts that matter):**

```java
public record EmailAddress(String value) {
  public static EmailAddress of(String raw) {
    var c = Text.normalize(raw);
    if (c == null || !c.matches("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"))
      throw new IllegalArgumentException("Invalid email");
    return new EmailAddress(c);
  }
}
public record Slug(String value) {
  public static Slug of(String raw) {
    var c = Text.normalize(raw).toLowerCase()
      .replaceAll("[^a-z0-9-]+","-").replaceAll("^-+|-+$","");
    if (c.isBlank() || c.length() > 80) throw new IllegalArgumentException("Invalid slug");
    return new Slug(c);
  }
}
```

**4) Service signatures take typed inputs (internal guard):**

```java
public record CreateCategory(Slug slug, EmailAddress ownerEmail, String description) {}

@Service
class CategoryService {
  public Category create(CreateCategory cmd) { /* cmd fields already canonical */ }
}
```

**5) DB guard:**

```sql
CREATE UNIQUE INDEX uq_category_name_ci ON category (lower(name));
-- Match VARCHAR lengths to your DTO/VO limits.
```

---

## Decision rules (fast heuristics)

* **Is this field free-text (description, notes)?**
  Normalize at endpoint only. Keep it a `String`. Don’t over-constrain.
* **Will this field be compared, deduped, or displayed in canonical form (email, slug, code)?**
  Make a **VO** for it. Services consume the VO. Add a DB index on a canonical expression.
* **Prototype vs hardening:**
  Start with service-level normalize for key fields → later flip on global Jackson normalizer and introduce VOs where rules matter. No breaking API changes.

---

## The elevator pitch you can put in your vault

> **Two gates, one truth.**
> Normalize at the **edge** for UX and consistent validation.
> Enforce invariants in the **domain** with reusable value objects so every caller—REST, CLI, Kafka—must pass through the same rules.
> Start with endpoint-only while learning; go full pipeline for production.
> Back it with DB constraints to catch races.

---

## Next steps to lock it in

1. Add `Text.normalize(...)`.
2. Add global Jackson String normalizer + `@Raw`.
3. Keep DTO validation.
4. Add a unique index on canonical name/slug in SQL.
5. When a field’s rules start to matter across the system, promote it to a VO and switch your service signature to accept that VO.

That’s the lean path: fast today, safe tomorrow, no rewrites.
