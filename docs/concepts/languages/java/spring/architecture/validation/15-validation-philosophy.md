---
title: Validation Philosophy
date: 2025-11-02
tags: 
    - java
    - spring
    - validation
    - architecture
summary: A deep dive into the philosophy of validation in Spring applications, exploring how validation shapes API design, DTOs, and overall application architecture for robust and maintainable code.
aliases:
  - Validation Philosophy
---


# Subtle but important truth

Validation isn't just *catching wrong input*.
It shapes **how you design your public surface**.

Your API contract is basically a promise to the world:

> “You send me a thing shaped like this, and I will behave predictably.”

Validation is the contract guardian.

When you treat validation as a first-class citizen, you nudge yourself toward:

* precise DTOs (each one with a single job)
* clear domain boundaries
* predictable failures
* fewer weird edge-case bugs
* better API documentation (Swagger loves constraints)

These aren’t small wins. They compound.

---

### DTO design philosophy

A DTO is not a “bag of fields”.
It’s a **request intention container**.

Bad mindset:
“User has name, email, password… so DTO has those fields.”

Better mindset:
“This request’s *intent* is to create a user. What fields and rules define *this action*?”

That's why `CreateUserRequest` and `UpdateUserRequest` are different beasts.

Create might require everything.
Update might accept only what changes.

Beginners often smash them together.
Seniors separate them — it pays off when complexity grows.

---

### When to validate at controller vs service

Controller validation (Jakarta) handles **input shape sanity**.

Service/domain should handle meaning:

Example:

```java
public User createUser(CreateUserRequest req) {
    if (emailExists(req.email())) {
        throw new EmailAlreadyUsedException(req.email());
    }
    if (calculateAge(req.birthDate()) < 18) {
        throw new UnderageException();
    }
    // valid meaning, continue
}
```

Input validity ≠ business validity.

If someone tells you “we validated everything already in DTO annotations, so service doesn't need checks,” congratulations — you’ve found a codebase destined for cryptic failures and existential bugs.

---

### Custom validators… advanced edition

Real validation often needs **context**.

Imagine:

* “Price must be > 0 **unless item is free**”
* “Password optional on update but strong if present”
* “Start date must be before end date — but only if range mode is enabled”
* “One of fields A, B, C must be provided, but not all three”

This leads to two patterns:

#### 1) Class-level validators

Used when validation depends on **multiple fields**.

```java
@ValidDateRange
public record BookingRequest(
    LocalDate start,
    LocalDate end
) {}
```

Validator inspects both values.

#### 2) Cross-field constraint logic in service

If the rule feels like business, not input shape, keep it in service.

Rule of thumb:
If your validator starts reading from a repository or external service, you're doing domain logic. Stop. Move to service.

Keep validators **stateless & pure** — like good monks of functional temple.

---

### Programmatic validation

Sometimes you need to validate an object **manually**.

Spring bean:

```java
@Autowired
Validator validator;
```

Use it inside service for cases like:

* validating objects created internally
* validating post-transformation values
* re-validating after filling defaults

```java
var result = validator.validate(obj);
if (!result.isEmpty()) {
    throw new InputIntegrityViolation(result);
}
```

Useful when your app mutates objects, hydrates defaults, etc.

---

### Performance notes (nerd corner)

Validation is cheap… but not free.

* Avoid massive DTOs.
* Avoid validators that query DB — that’s anti-pattern territory.
* Avoid recursive validation on giant collections by accident (yes, I’ve seen a 50k row CSV upload destroy a server via `@Valid` list).

Moral: Validate what humans submit. Not what batch jobs feed.

---

### Common beginner mistakes

Seen so many times it’s almost poetry:

* Putting `@NotNull` on primitives (they can’t be null — use wrapper types if needed)
* Using `@NotEmpty` instead of `@NotBlank` for strings (allows `"   "`)
* Validating DTO but never domain
* Writing a giant “one DTO to rule them all”
* Returning raw validator errors to API consumers — unreadable, unstructured chaos
* Forgetting `@Valid` on nested fields → silent failure
* Writing business rules inside validation annotations

Mistakes are fine. Staying in them isn’t.

---

### What good validation feels like

You hit `/users` with garbage input…

*Error message comes back structured*
*Fields are pointed out precisely*
*Client dev understands exactly what to fix*
*You don't chase phantom bugs in services*

The experience feels… designed.

This is what separates hobby projects from mature APIs.

---

### Where this threads into bigger backend craft

Jakarta Validation is the tip of a deeper spear:

* input correctness (validation)
* data normalization & sanitation
* API schemas (OpenAPI / Swagger)
* global error architecture (ProblemDetails etc.)
* domain invariants
* mapping consistency (DTO ↔ command ↔ domain)
* observability — knowing when input is garbage or attacker-shaped

All roads lead toward one goal:

> A backend that never accepts nonsense and never lies.

