---
title: Validation Messages
date: 2025-11-07
tags: 
    - java
    - spring
    - i18n
    - internationalization
    - bean-validation
    - cheatsheet
summary: A quick reference on integrating Bean Validation messages with internationalization (i18n) in Spring applications, covering best practices for externalizing validation messages into message bundles.
aliases:
    - Spring i18n - Validation Messages Cheatsheet
---



# Validation Messages — Bean Validation + i18n

Validation errors are where language leaks into your API most aggressively. Every constraint — `@NotBlank`, `@Email`, `@Size`, `@Pattern` — wants to talk to the outside world.  Hardcoding strings inside annotations works during tutorials; after that it becomes a linguistic minefield. 

Spring + Bean Validation + message bundles give you a single, stable way to express human-readable validation errors across locales. This page covers how.

---

## The key idea

Bean Validation annotations accept *keys*, not raw text.

```java
@NotBlank(message = "{user.email.required}")
```

Spring takes the `{…}` form as a lookup into `messages.properties`, applying locale resolution automatically.

That’s the whole trick — simple and powerful.

---

## The message bundle side

You define the messages here:

```
src/main/resources/messages.properties
src/main/resources/messages_en.properties
src/main/resources/messages_de.properties
```

Example:

```
user.email.required=Email is required.
user.email.invalid=Email is not valid.
user.name.required=Name cannot be blank.
```

This isolates the *meaning* from any particular language.

---

## When.annotations.use.key vs literal

**Bad (hardcoded):**

```java
@NotBlank(message = "Email is required")
```

Hard to change, impossible to localize.

**Better (key-based):**

```java
@NotBlank(message = "{user.email.required}")
```

This gives you:

* centralized message management
* multi-language support
* consistent vocabulary across your project
* testability (assert keys exist)

---

## Common constraints and how to externalize them

### `@NotBlank`

```java
@NotBlank(message = "{user.username.required}")
```

### `@Email`

```java
@Email(message = "{user.email.invalid}")
```

### `@Size`

```java
@Size(min = 3, max = 30, message = "{user.username.size}")
```

Bundle:

```
user.username.size=Username length must be between {min} and {max} characters.
```

Note: Bean Validation injects `{min}` and `{max}` automatically.

### Custom constraints

Custom validator:

```java
@ValidCategorySlug(message = "{category.slug.invalid}")
```

Bundle:

```
category.slug.invalid=Category slug contains illegal characters.
```

---

## Attribute interpolation (the good magic)

For size, range, digits, etc., Bean Validation exposes attributes as variables:

```java
@Size(min = 2, max = 50, message = "{category.name.length}")
```

Bundle:

```
category.name.length=Length must be between {min} and {max}.
```

This makes messages expressive without repetition.

---

## How validation integrates with your web edge

Spring’s `MethodArgumentNotValidException` hits your global exception handler.
Your handler extracts:

* field name
* rejected value
* the resolved, localized message

Then you wrap it into a predictable, RFC-7807-style `ProblemDetail`:

```java
problemDetail.setProperty("errors", List.of(
  Map.of(
    "field", "email",
    "message", messages.getMessage(errorCode, errorArgs, locale)
  )
));
```

This is where localized validation messages flow directly into the `errors` section of ProblemDetail.

Your API becomes multilingual without rewriting your handler logic.

---

## When to centralize vs when to split bundles

Start with one file.

Split only when one of these happens:

* validation messages drown out domain errors
* you build reusable modules (e.g., user module with its own bundle)
* your bundles exceed 300–400 lines
* you want very different lifecycle management (e.g., validation messages rarely change, but business messages change often)

With multiple bundles:

```java
source.setBasenames(
  "classpath:messages",
  "classpath:messages/validation"
);
```

---

## Naming conventions — keep the semantics tight

A battle-tested style:

```
<domain>.<entity>.<field>.<meaning>
```

Examples:

```
user.email.required
user.email.invalid
user.password.weak
category.name.required
category.slug.invalid
product.price.negative
```

Use uppercase/literals **only** for constants or codes; keys stay lowercase with dots.

This gives you a dictionary that scales with your domain.

---

## Testing validation messages

Integration tests confirm both validation *and* message resolution:

```java
mockMvc.perform(post("/users")
    .content("{\"email\": \"\"}")
    .contentType(APPLICATION_JSON))
  .andExpect(status().isBadRequest())
  .andExpect(jsonPath("$.errors[0].message").value("Email is required."));
```

This is the real safety net:
your localization can’t drift out of sync unnoticed.

---

## Validation vs domain errors vs system errors

Validation messages belong to:

* request DTOs
* field-level rules
* shape & constraints of the outside world

They **don’t** belong to:

* domain invariants
* business rules
* internal failures

Those get their own keys and usually land in a different section of your bundle (or a separate file entirely).

Understanding this boundary keeps your messages logically structured as your app grows.

---

## This page unlocks the final i18n piece

With this glued together, the next natural file is:

**`locale-switching.md`**
which explains *how* Spring decides which language bundle to use for the message keys you define here.

This completes i18n triad:

1. **messages-basics** — the mechanics
2. **validation-messages** — the Bean Validation bridge
3. **locale-switching** — the language decision logic

Once all three are in place, your i18n setup becomes production-ready.
