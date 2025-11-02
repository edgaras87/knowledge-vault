---
title: What is Jakarta Validation?  
date: 2025-11-02
tags: 
    - java
    - spring
    - validation
    - architecture
    - jakarta
summary: A concise overview of Jakarta Validation (formerly javax.validation) in Spring applications, explaining its purpose, common constraints, custom validators, and best practices for use in application architecture.
aliases:
  - What is Jakarta Validation?
---



# What *is* Jakarta Validation?

Jakarta Validation (formerly `javax.validation` → now `jakarta.validation`) is a specification and API for declaring **constraints on object fields**, usually using annotations, and validating them automatically.

You write rules like:

```java
public record CreateUserRequest(
    @NotBlank @Email String email,
    @NotBlank String name,
    @Size(min = 8) String password
) {}
```

And your framework goes:

> “Alright, if someone tries to POST a user without an email or with a cat-length password, we refuse politely.”

It’s declarative validation. You’re describing rules. You’re not writing `if (email == null || email.isBlank()) throw new ...`.

That saves you from “spaghetti validation hell” — those ugly forests of `if`s that creep into beginner service layers.

---

### Why does it exist?

Historically, developers scattered validation logic all over:

* UI input checks
* Controller checks
* Service layer checks
* Database constraints

Each spot duplicated the same rules. Boredom, bugs, sadness.

Jakarta Validation centralizes rules so you can:

* Declare once, validate everywhere
* Keep your business code focused on business logic, not guarding doors
* Get consistent error format for bad input
* Plug in custom validators when life gets fancy

The universe likes DRY code; this is a tool aligned with entropy reduction.

---

### How does Spring hook into it?

Spring uses Hibernate Validator under the hood to implement the Jakarta spec. When you put `@Valid` in your controller method, you're saying:

> "Framework, inspect this object, and if it doesn't meet expectations, toss a polite tantrum."

Example:

```java
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(
        @Valid @RequestBody CreateUserRequest req) {
    ...
}
```

When validation fails, Spring throws `MethodArgumentNotValidException`. If you have a global exception handler, you translate that into your structured API error.

Without one, Spring returns its default JSON scolding.

---

### Common constraints you’ll see

Human-memory-friendly version first, so your brain absorbs the essence:

* **@NotNull** — must exist
* **@NotBlank** — must contain non-space characters
* **@Email** — must smell like an email
* **@Size(min, max)** — good for strings & collections
* **@Min / @Max** — number guardians
* **@Pattern** — regex wizardry
* **@Past / @Future** — great for dates (birthday in the future? you’re time-traveling or lying)

Mechanical versions, if your cortex enjoys officiality:

```java
@NotNull
@NotBlank
@NotEmpty
@Email
@Size(min = 3, max = 255)
@Min(1)
@Max(100)
@Positive
@Negative
@PositiveOrZero
@Pattern(regexp="regex here")
@Past
@PastOrPresent
@Future
@FutureOrPresent
```

You will forget `@NotEmpty` exists because `@NotBlank` is nicer for text. This is fine. The world continues spinning.

---

### Beyond simple annotations: custom validators

Sometimes reality is annoying, like validating:

* Password strength (uppercase, digit, symbol…)
* "End date must be after start date"
* "Username cannot contain the word 'bot'"
* "VAT number valid for EU country"

You create annotation + validator:

```java
@Target({ FIELD })
@Retention(RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
public @interface StrongPassword {
    String message() default "Weak password";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

Implement logic:

```java
public class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        return value != null
               && value.matches(".*[A-Z].*")
               && value.matches(".*\\d.*")
               && value.length() >= 8;
    }
}
```

These become reusable, clean, and readable.

Just don't get too validator-happy and try encoding your whole business logic there. Validators are for *input sanity*, not *domain decisions*.

---

### Where validation belongs in the grand architecture story?

Quick conceptual map:

* **DTO / Request layer** → use Jakarta Validation (describing *shape of input*)
* **Domain layer** → enforce *invariants* (business rules)
* **DB** → final sanity gates (e.g., unique constraints)

Think of it like concentric shields:
client input → request validation → domain invariants → database constraints

Validation does not replace business rules. Your domain still says things like:

> "User must be at least 18 to register."

Validation only says:

> "Age field must be present and a number."

---

### Future-proof note: `javax.validation` vs `jakarta.validation`

Modern apps = `jakarta.validation`. Old tutorials may still show `javax.*`. Same concept, just a namespace shift when Java EE moved to Eclipse Foundation.

If your IDE screams mismatched imports, it's not angry; it's reminding you time moves forward.

---

### Quick example end-to-end

Request DTO:

```java
public record CreateProductRequest(
    @NotBlank String name,
    @Min(1) int quantity,
    @Positive double price
) {}
```

Controller:

```java
@PostMapping
public ResponseEntity<ProductResponse> create(
        @Valid @RequestBody CreateProductRequest req) {
    var result = service.create(req);
    return ResponseEntity.ok(result);
}
```

If validation fails, your global handler formats it to ProblemDetail or your flavor of JSON errors.

Clean, predictable, civilized.

---

### Mental model to carry

Jakarta Validation is not a silver bullet. It’s a first line of defense, not your whole army. It keeps nonsense out, but meaning is enforced by your business logic.

Same way a gym membership won't make you fit — but it definitely helps.

---

### Where to explore next

Once you’re comfortable with validation, interesting paths branch out:

* ProblemDetail + error catalog patterns
* Constraint composition (`@ValidAge @ValidName`)
* Validation groups (e.g. different rules for CREATE vs UPDATE)
* Record DTOs vs JavaBeans
* Fluent error reporting patterns
* Domain invariants vs input validation boundaries

Validation is just the first guard tower. The castle of backend design keeps going.

