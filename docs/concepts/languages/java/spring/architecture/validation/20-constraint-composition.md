---
title: Constraint Composition 
date: 2025-11-02
tags: 
    - java
    - spring
    - validation
    - architecture
summary: How to compose validation constraints in Jakarta Validation for cleaner and more maintainable code in Spring applications, including custom constraint annotations, best practices, and common pitfalls.
aliases:
  - Constraint Composition
---

# Constraint Composition — building your own language of rules

Eventually you get bored of repeating annotations:

```java
@NotBlank
@Email
@Size(max = 255)
String email
```

Feels like writing out your shopping list every day.

Instead, bundle constraints into a single annotation:

```java
@Target({ FIELD, PARAMETER })
@Retention(RUNTIME)
@Constraint(validatedBy = {})
@NotBlank
@Email
@Size(max = 255)
public @interface ValidEmail {
    String message() default "Invalid email format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

Now your DTO becomes zen-like:

```java
public record CreateUserRequest(
    @ValidEmail String email,
    @NotBlank String name
) {}
```

Custom DSL for data correctness. Much cleaner. Much more “enterprise samurai”.

---

### The dark arts — when validation goes wrong

Truth time: validation *can* become a trap if abused.

Smells to watch for:

**1. Validators querying the database**
Using repository inside validator = slow, brittle, and merges layers wrongly.

Flag it immediately. Put uniqueness checks in services.

**2. Over-validating**
Example: enforcing domain business rules inside DTOs.

Input validation should describe **shape**, not **policy**.

**3. Reusing the same DTO everywhere**
Common rookie move:

* They use `UserDto` for create, update, API response, internal logic.
* Eventually it gains 30 fields and 20 annotations and becomes a Lovecraftian monster.

Solution: separate DTOs by intent, not ego.

---

### Spring’s validation pipeline — what happens under the hood

When you do:

```java
public ResponseEntity<?> create(@Valid @RequestBody UserInput req)
```

Flow looks like:

1. JSON gets deserialized
2. Spring detects `@Valid`
3. Hibernate Validator runs constraints
4. If violations found → throws `MethodArgumentNotValidException`
5. Your `@RestControllerAdvice` catches and formats it
6. Controller logic runs only if clean

Important realization:
**Bad input never reaches your service** (if you architect correctly).

That’s not just safety — it’s clarity. Your core logic wakes up only for sane requests.

---

### Error messages — human dignity matters

Try this:

```java
@NotBlank(message = "Name is required") String name;
```

instead of:

```java
@NotBlank(message = "must not be blank")
```

One talks to a machine.
The other talks to a person.

Good API designers think about tone — even in errors.

---

### i18n — multilingual validations

You can externalize messages into `messages.properties`:

```
user.name.notBlank=Name is required.
```

In annotation:

```java
@NotBlank(message = "{user.name.notBlank}")
String name;
```

This becomes priceless if your client base includes people who speak more than one language or one developer in Brazil and another in Lithuania (wink).

---

### Order matters — validation sequence

Jakarta lets you enforce validation order:

* first check null
* then pattern
* then domain logic

You define groups and sequence:

```java
@GroupSequence({Step1.class, Step2.class, Step3.class})
public interface ValidationOrder {}
```

Rarely used early in career but deeply helpful in real multi-step workflows.

---

### Immutable DTOs vs validation — an interesting tension

Java records are clean and immutable-ish.
Validation framework expects mutable bean style?
Not exactly — it works fine.

But two patterns emerge:

**Constructor normalization** (good):

```java
public record UserRequest(
    @NotBlank String name
) {
    public UserRequest {
        name = name.trim();
    }
}
```

**Mutating after validation** (bad):

```java
req.setName(req.getName().trim());
```

Try not to mutate validated data. Feed it forward as-is or revalidate.

Immutability + validation = happy data pipeline.

---

### You don't validate *everything*

Say your system generates timestamps internally.

Do you put `@PastOrPresent` on them?
No. You trust your system.

Validation gates untrusted input from outside your core.
Inside? You enforce rules via code and domain objects.

That separation is elegance.

---

### Security — the silent twin

Validation ≠ security.
But validation feeds security.

Example:

* `@Pattern` helps avoid command injection
* normalizing Unicode avoids homoglyph attacks
* size limits prevent denial-of-service payloads

Combine:

| Concern           | Tool                    |
| ----------------- | ----------------------- |
| Input format      | Jakarta Validation      |
| Input size & rate | Filters / rate limiters |
| Auth / access     | Security layer          |
| Domain truth      | Business rules          |

Validation is one shield, not the castle.

---

### Zooming out — why this really matters

Early on, validation feels like checklist work.
Later you realize it shapes:

* API reliability
* client happiness
* system resilience
* developer sanity
* future refactoring ease

Bad data breaks systems quietly.
Good validation prevents the tragedy.

It’s like washing your hands in a hospital — boring until you skip it and the whole ICU collapses.

---

### Where we go next (options)

Pick any future branch:

* Building a reusable **problem-details error handler**
* Trimming & canonicalization strategies
* DTO vs Command vs Domain entity — strict separation
* Validation for PATCH requests (partial updates)
* Nested error reporting patterns
* How to enforce domain invariants cleanly
* When validation moves to message queues (Kafka events, etc.)
* Testing validation — unit vs integration
* Real-world “validation + security” pitfalls

Each topic adds another brick in the wall of real backend mastery — the kind you feel in your fingers when writing code, not the kind textbooks preach.

