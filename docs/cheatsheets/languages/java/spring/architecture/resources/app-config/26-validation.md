---
title: Validation
date: 2025-11-06
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for Spring Boot property validation, detailing how to enforce constraints on configuration properties using Jakarta Bean Validation annotations.
aliases:
  - Spring Properties - Validation Cheatsheet
---

# Property Validation — Cheatsheet

Property validation ensures your application fails fast when configuration is invalid  
(wrong host, negative limits, missing secrets).  
Spring integrates Jakarta Bean Validation with `@ConfigurationProperties` so the app refuses to start if values don’t meet constraints.

---

## 1. Enabling validation

To validate a configuration class, annotate it with `@Validated`.

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProps(
  @NotNull URI host,
  @Min(1) Duration timeout,
  Limits limits
) {
  public record Limits(
    @Min(1) int itemsPerPage,
    @NotNull DataSize uploads
  ) {}
}
```

Make sure Bean Validation is on the classpath (Spring Boot includes it by default).

---

## 2. How it behaves at startup

If a constraint fails, the application stops booting:

```
Binding to target AppProps failed:

  Property: app.limits.itemsPerPage
  Value: 0
  Reason: must be greater than or equal to 1
```

This is intentional — invalid configuration is a *startup error*, not a runtime bug.

---

## 3. Common annotations (for config)

### @NotNull  
Required field must be provided.

### @NotBlank / @NotEmpty  
For non-empty Strings or collections.

### @Min / @Max  
Numeric constraints.

### @Positive / @PositiveOrZero  
For values like timeouts, limits, pool sizes.

### @Pattern  
For basic string format validation (hostnames, simple emails, IDs).

### @DurationMin / @DurationMax  
Spring Boot provides these for `Duration`:

```java
@DurationMin(seconds = 1)
Duration timeout
```

### Custom validator  
You can build domain checks like “must be a valid URL base”:

```java
@ValidHostBase
URI host;
```

---

## 4. Validating nested structures

Validation works recursively:

```java
public record AppProps(
  @Valid Limits limits
) {
  public record Limits(
    @Min(1) int itemsPerPage
  ) {}
}
```

The `@Valid` signals Spring to descend into nested objects/records.

Records usually don’t need `@Valid` if they’re detected automatically,  
but adding it is never harmful and improves clarity.

---

## 5. Defaults + validation

If you combine defaults with validation:

```java
public record AppProps(Duration timeout) {
  public AppProps {
    if (timeout == null) timeout = Duration.ofSeconds(5);
  }
}
```

Then apply:

```java
@DurationMin(seconds = 1)
Duration timeout
```

If YAML omits the value, the default is used.  
If YAML overrides with a smaller value, validation kicks in.

---

## 6. When to validate, when not to

**Validate when:**
- A value is required for the app to run.
- A numeric bound prevents misconfiguration (uploads, pool sizes).
- A URL/URI must follow a specific form.
- A feature flag must be boolean, not string.

**Do not validate when:**
- The value is optional and your code can handle `null` gracefully.
- The shape is not fixed (e.g., flexible user-defined mappings).
- The config is only used in local development.

---

## 7. Validation errors in tests

When testing a configuration class:

```java
@SpringBootTest
@EnableConfigurationProperties(AppProps.class)
@TestPropertySource(properties = {
  "app.timeout=0s"   # invalid
})
class InvalidPropsTest {

  @Test
  void failsFastOnInvalidConfig() {
    assertThrows(Exception.class, () -> { /* context loads */ });
  }
}
```

Or isolate using:

```java
@Import(AppProps.class)
```

If you want the test context to fail loudly, keep `@SpringBootTest`.  
If you want isolated tests, use `ApplicationContextRunner` from Spring Boot.

---

## 8. Applying validation to @Value (not recommended)

Technically possible:

```java
@Value("${app.items}")
@Min(1)
int items;
```

But not recommended:

- Errors surface *after* the bean is created.
- Harder to test.
- No structure.

Prefer `@ConfigurationProperties`.

---

## 9. Practical example matching your AppProps

**YAML**

```yaml
app:
  host: http://localhost:8080
  limits:
    uploads: 20MB
    itemsPerPage: 0   # invalid
```

**Java**

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProps(
  @NotNull URI host,
  @Valid Limits limits
) {
  public record Limits(
    @Valid @DataSizeMin("1MB") DataSize uploads,
    @Min(1) int itemsPerPage
  ) {}
}
```

**Startup fails**, reporting the violation with the exact key path.

---

## 10. DO / DON'T / pitfalls

**DO**
- Keep config immutable (records).
- Validate only what matters for safety.
- Validate nested structures when they exist.
- Give YAML defaults for “happy path” values.

**DON'T**
- Don’t put runtime logic into validation.
- Don’t validate secrets (just ensure they exist).
- Don’t rely on @Value + validation.
- Don’t create deeply nested config; keep domain small and focused.

**Pitfalls**
- Forgetting `@Validated` → no validation happens.  
- Forgetting `@Valid` on nested structures → child fields not validated.  
- Using `@NotBlank` for optional fields → app fails even if null was fine.

---

## 11. Quick glossary

- **Bean Validation (Jakarta)** — standard for constraints (`@Min`, `@NotNull`, etc.).  
- **Fail-fast** — Spring refuses to start if validation fails.  
- **Nested validation** — recursive validation through `@Valid`.  
- **Constraint violations** — messages explaining why binding failed.

