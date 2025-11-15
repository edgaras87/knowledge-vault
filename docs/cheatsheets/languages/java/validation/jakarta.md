---
title: Jakarta
date: 2025-11-15
tags: 
  - java
  - validation
  - jakarta 
  
summary: A concise cheatsheet for Jakarta Validation (Bean Validation) in Java, covering common constraints, usage in Spring Boot, and custom validations.
aliases:
  - Jakarta Validation — Cheatsheet
---

# Jakarta Validation — Cheatsheet

Package: `jakarta.validation.constraints.*`

Validation defines **rules for incoming data**: DTO fields, config properties, method parameters.
It does **not** touch the database.
Think of it as the “first guardrail” at your app’s web edge.

---

# 0. Quick Summary Table (with short explanations + links)

| Annotation                          | What It Does                                       | More                                               |
| ----------------------------------- | -------------------------------------------------- | -------------------------------------------------- |
| **@NotNull**                        | Must not be null (but can be empty)                | → [Section 1.1](#11-notnull)                       |
| **@NotBlank**                       | Must not be null/empty/whitespace (String only)    | → [Section 1.2](#12-notblank)                      |
| **@NotEmpty**                       | Not null & not empty (String/Collection/Map/Array) | → [Section 1.3](#13-notempty)                      |
| **@Size(min,max)**                  | Length/size bounds                                 | → [Section 1.4](#14-size)                          |
| **@Email**                          | Basic email format validation                      | → [Section 1.5](#15-email)                         |
| **@Pattern(regexp)**                | Regex match                                        | → [Section 1.6](#16-patternregexp)                 |
| **@Positive** / **@PositiveOrZero** | Numeric >0 / ≥0                                    | → [Section 1.7](#17-positive--positiveorzero)      |
| **@Negative** / **@NegativeOrZero** | Numeric <0 / ≤0                                    | → [Section 1.8](#18-negative--negativeorzero)      |
| **@Min(n)** / **@Max(n)**           | Numeric boundaries                                 | → [Section 1.9](#19-min--max)                      |
| **@Digits(integer,fraction)**       | Decimal formatting constraints                     | → [Section 1.10](#110-digitsinteger-fraction)      |
| **@Past / @PastOrPresent**          | Date/time must be before now                       | → [Section 2.1](#21-past--pastorpresent)           |
| **@Future / @FutureOrPresent**      | Date/time must be after now                        | → [Section 2.2](#22-future--futureorpresent)       |
| **@AssertTrue / @AssertFalse**      | Boolean logic validation                           | → [Section 3.1](#31-asserttrue--assertfalse)       |
| **@Valid**                          | Cascade validation into nested objects             | → [Section 4.1](#41-valid)                         |
| **Container element constraints**   | Validate each element of a list/map                | → [Section 4.2](#42-container-element-constraints) |
| **Custom @Constraint(...)**         | Your own business rule                             | → [Section 8](#8-custom-constraints)               |

---

# 1. Core Annotations (Your Daily Tools)

---

## 1.1 `@NotNull`

Value must not be `null`.
Does **not** check emptiness.

```java
@NotNull
Integer age;
```

→ Works on any type.

---

## 1.2 `@NotBlank`

String must not be null/empty/whitespace.

```java
@NotBlank
String username;
```

---

## 1.3 `@NotEmpty`

Not null and not empty.
Works for String, Collection, Map, Array.

```java
@NotEmpty
List<Long> ids;
```

---

## 1.4 `@Size`

Length/size constraints.

```java
@Size(min = 1, max = 100)
String title;

@Size(max = 10)
List<String> tags;
```

---

## 1.5 `@Email`

Basic email format.

```java
@Email
String email;
```

Custom domain:

```java
@Email(regexp = ".*@example\\.com")
```

---

## 1.6 `@Pattern(regexp)`

Regex match.

```java
@Pattern(regexp = "^[a-z0-9-]{3,50}$")
String slug;
```

---

## 1.7 `@Positive` / `@PositiveOrZero`

Greater than zero / greater or equal to zero.

```java
@Positive
Integer quantity;
```

---

## 1.8 `@Negative` / `@NegativeOrZero`

```java
@Negative
int offset;
```

---

## 1.9 `@Min` / `@Max`

Numeric bounds.

```java
@Min(18)
Integer age;
```

---

## 1.10 `@Digits(integer, fraction)`

Decimal formatting.

```java
@Digits(integer = 8, fraction = 2)
BigDecimal price;
```

---

# 2. Date & Time Constraints

---

## 2.1 `@Past` / `@PastOrPresent`

```java
@Past
LocalDate birthDate;
```

---

## 2.2 `@Future` / `@FutureOrPresent`

```java
@Future
Instant scheduledAt;
```

---

# 3. Boolean Logic Constraints

---

## 3.1 `@AssertTrue` / `@AssertFalse`

```java
@AssertTrue
boolean agreed;
```

Often replaced by explicit booleans in DTOs.

---

# 4. Composite / Nested Validation

---

## 4.1 `@Valid`

Enables validation inside nested objects.

```java
public record CreateOrderRequest(
  @Valid AddressRequest address
) {}
```

---

## 4.2 Container Element Constraints

Validates each element in collection/map.

```java
List<@Email String> emails;
Map<String, @Positive Integer> scores;
```

---

# 5. Full Number Constraint Table

| Annotation        | Meaning                                 |
| ----------------- | --------------------------------------- |
| `@Min(n)`         | ≥ n                                     |
| `@Max(n)`         | ≤ n                                     |
| `@Positive`       | > 0                                     |
| `@PositiveOrZero` | ≥ 0                                     |
| `@Negative`       | < 0                                     |
| `@NegativeOrZero` | ≤ 0                                     |
| `@Digits(i,f)`    | i digits before decimal, f digits after |

---

# 6. Full String Constraint Table

| Annotation  | Meaning                             |
| ----------- | ----------------------------------- |
| `@NotBlank` | not null, not empty, not whitespace |
| `@NotEmpty` | not null, not empty                 |
| `@Size`     | min/max length                      |
| `@Pattern`  | regex                               |
| `@Email`    | email format                        |

---

# 7. Method-Level & Bean-Level Validation

You can validate **derived logic**:

```java
@AssertTrue(message = "End date must be after start date")
public boolean isValidRange() {
  return endDate.isAfter(startDate);
}
```

---

# 8. Custom Constraints

Define your own annotation:

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = SlugValidator.class)
public @interface ValidSlug {
  String message() default "Invalid slug";
  Class<?>[] groups() default {};
  Class<?>[] payload() default {};
}
```

Validator:

```java
public class SlugValidator implements ConstraintValidator<ValidSlug, String> {
  public boolean isValid(String value, ConstraintValidatorContext ctx) {
    return value != null && value.matches("^[a-z0-9-]{3,50}$");
  }
}
```

---

# 9. Practical Best Practices

**Do:**

* Validate DTOs, *not* entities.
* Combine:

```java
@NotBlank
@Size(max = 100)
String name;
```

* Use container element validation:

```java
List<@Positive Long> ids;
```

**Avoid:**

* Using regex where type already enforces structure.
* Validating persistence entities.

---

# 10. Quick Reference Summary

* **Null rules** → `@NotNull`, `@NotEmpty`, `@NotBlank`
* **Length rules** → `@Size`
* **Format rules** → `@Pattern`, `@Email`
* **Numeric rules** → `@Positive`, `@Min`, etc.
* **Date rules** → `@Past`, `@Future`
* **Nested validation** → `@Valid`
* **Cross-field logic** → `@AssertTrue`
* **Custom rules** → `@Constraint`, + validator class

