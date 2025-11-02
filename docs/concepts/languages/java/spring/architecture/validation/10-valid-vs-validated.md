---
title: @Valid vs @Validated 
date: 2025-11-02
tags: 
    - java
    - spring
    - validation
    - architecture
summary: Understanding the differences between `@Valid` and `@Validated` in Spring applications, including when to use each, validation groups, nested validation, and best practices for effective input validation.
aliases:
  - @Valid vs @Validated
---



# `@Valid` vs `@Validated`

Two cousins at the cookout. Both look similar, but one shows off party tricks.

**`@Valid`**
From Jakarta Validation. It says:

> “Validate this object using default constraints.”

You slap it on method parameters, nested fields, collections, etc.

Example:

```java
public ResponseEntity<?> create(@Valid @RequestBody UserInput input) { ... }
```

Neat. Straightforward. No spice, just salt.

**`@Validated`**
From Spring. Think of it as `"@Valid + groups support"`.

```java
@Validated(Create.class)
public ResponseEntity<?> create(@RequestBody UserInput input) { ... }
```

Now you can choose which rules apply *based on context*.

Use `@Validated` when you need:

* different constraints for create vs update
* partial validation
* grouping heavy business flows

Use `@Valid` when life is simple and sunshiney.

---

### Validation Groups — situational sanity checks

Sometimes rules change based on operation:

* Creating user: `@NotBlank name`
* Updating user: `name` optional (PATCH / PUT partial inputs)

Groups solve it elegantly.

Define marker interfaces:

```java
public interface Create {}
public interface Update {}
```

Modify constraints:

```java
public record UserInput(
    @NotBlank(groups = Create.class)
    String name,

    @Email(groups = {Create.class, Update.class})
    String email
) {}
```

Controller chooses group:

```java
@PostMapping
public void create(@Validated(Create.class) @RequestBody UserInput input) {}
```

For update:

```java
@PatchMapping("/{id}")
public void update(@Validated(Update.class) @RequestBody UserInput input) {}
```

Clean. Predictable. No brittle “if field != null then validate”.

---

### Nested validation and why `@Valid` matters on fields

If your DTO nests other DTOs, you need `@Valid` on the field:

```java
public record OrderRequest(
    @NotNull Long userId,
    @Valid Address address
) {}

public record Address(
    @NotBlank String street,
    @NotBlank String city
) {}
```

Without `@Valid` on `address`, the validator sees it as a mysterious blob and lets bad stuff slip through. Feels like locking your front door but leaving the windows open.

---

### Collections vs Elements

Validating a list of objects? Same idea:

```java
public record BatchCreateUsersRequest(
    @NotEmpty
    @Valid List<CreateUserRequest> users
) {}
```

This avoids the classic nightmare of “first user is valid, second user is a chaos gremlin”.

---

### Sanitizing input ≠ validating input

Validation:
“Is your email shaped like an email?”

Sanitization:
“Did you paste three spaces and an invisible Unicode goblin before your email?”

Trim input early. Normalize text. Kill sneaky whitespace. Anti-gremlin hygiene.

```java
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {

    public CreateUserRequest {
        name = name == null ? null : name.trim();
        email = email == null ? null : email.trim().toLowerCase();
    }
}
```

Records handle this transformation beautifully.

---

### Domain rules still matter

Jakarta Validation checks **form**.
Your domain checks **meaning**.

Validation might ensure `age >= 1`.
Domain says `age >= 18 to open account`.

Never put domain rules in validation annotations. That’s like putting philosophy debates in your front-door peephole.

---

### BindingResult — be cautious

Spring gives you this:

```java
public ResponseEntity<?> create(
    @Valid @RequestBody UserInput input,
    BindingResult errors
) { ... }
```

It catches errors **manually**.
Most beginners stumble into using it everywhere and build “if (errors.hasErrors())” jungles.

Reality:
In clean architecture, you rarely touch it. Let your GlobalExceptionHandler do the heavy lifting.

Use BindingResult only when:

* you need **partial acceptance**
* you’re building a UI form flow (old-school MVC)
* you want to collect warnings, not just errors

REST APIs don’t gulp this often.

---

### When Jakarta Validation *is not enough*

Cases where annotations can't save you:

* Ensure email is unique → service check
* Balance cannot go negative → domain rule
* `endDate > startDate` → custom validator or domain logic
* Rate limit rules, business workflows, external system constraints

Validation is a shallow filter.
Domain logic is the truth engine.

---

### Error Responses — the adulting stage

Once you validate, you must *communicate failure well*.

Good REST returns structured ProblemDetail:

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation failed",
  "status": 400,
  "errors": {
    "email": "must be a valid email",
    "name": "must not be blank"
  }
}
```

This isn't fluff. Strong error communication = easier clients, cleaner debugging, happier future you.

---

### Next steps on the staircase

Direction for deeper confidence:

* explore Constraint composition
* write 2–3 custom validators
* implement validation groups in a real CRUD flow
* integrate with ProblemDetail in Spring
* practice nested and list validation
* input trimming and normalization everywhere
* avoid pushing domain logic into annotations — keep boundaries sharp

Backend is a castle. Validation is the moat.
Business rules are the stone walls.
The database is the final iron gate that slams shut on nonsense.

