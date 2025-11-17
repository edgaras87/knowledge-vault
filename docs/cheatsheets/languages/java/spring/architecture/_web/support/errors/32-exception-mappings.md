---
title: ExceptionMappings 
date: 2025-11-08
tags:
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A quick reference on managing exception-to-problem mappings in Spring applications using a centralized ExceptionMappings registry for consistent HTTP status and error codes in RFC-7807 problem responses.
aliases:
    - Spring Web Support Errors Layer - ProblemFactory Cheatsheet
---

# ExceptionMappings — Machine Registry for Problems

`ExceptionMappings` is the one place that knows the **HTTP status** and machine **code** for each problem **slug**.  
Optionally, it also maps exception **types** to **slugs** so your `@ControllerAdvice` can be almost logic-free.

- Slug → `Meta(status, code)` (e.g., `resource-not-found → (404, NOT_FOUND)`)
- Exception class → slug (e.g., `UserNotFoundException → resource-not-found`)
- Safe lookups (`meta(slug)`) for `ProblemFactory`

---

## Where it lives

- **Class:** `com.example.app.web.support.errors.ExceptionMappings`
- **Used by:** `ProblemFactory`, `GlobalExceptionHandler`
- **Depends on:** nothing (pure data/registry)

---

## Design notes

- Keep the **human text** out of here (that belongs to `ProblemText` via `MessageSource`).
- Keep **identity** (slug) stable. Status or wording can evolve without breaking clients.
- Provide simple **programmatic extension** points so features can add their own mappings.

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/errors/ExceptionMappings.java
package com.example.app.web.support.errors;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

import static com.example.app.web.support.errors.ProblemCatalog.*;

@Component
public class ExceptionMappings {

  /** Machine meta for a problem slug. */
  public record Meta(HttpStatus status, String code) {
    public Meta {
      Objects.requireNonNull(status, "status");
      Objects.requireNonNull(code, "code");
    }
  }

  // slug → meta (status, code)
  private final ConcurrentHashMap<String, Meta> bySlug = new ConcurrentHashMap<>();

  // exception class (FQN) → slug (optional convenience)
  private final ConcurrentHashMap<String, String> exToSlug = new ConcurrentHashMap<>();

  public ExceptionMappings() {
    // --- seed defaults (adjust to your catalog) ---
    putSlug(RESOURCE_NOT_FOUND, new Meta(HttpStatus.NOT_FOUND, "NOT_FOUND"));
    putSlug(VALIDATION_FAILED,  new Meta(HttpStatus.BAD_REQUEST, "VALIDATION_FAILED"));
    putSlug(MALFORMED_REQUEST,  new Meta(HttpStatus.BAD_REQUEST, "MALFORMED_REQUEST"));
    putSlug(INTERNAL_ERROR,     new Meta(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"));

    // example exception → slug pairs (register your own in feature config)
    // putException(UserNotFoundException.class, RESOURCE_NOT_FOUND);
  }

  // ---------- public API ----------

  /** Lookup machine meta for a slug, or null if not present. */
  public Meta meta(String slug) {
    return bySlug.get(slug);
  }

  /** Lookup a slug for the given exception type; walks the class hierarchy (first hit wins). */
  public Optional<String> slugFor(Throwable ex) {
    if (ex == null) return Optional.empty();
    Class<?> c = ex.getClass();
    while (c != null && c != Object.class) {
      String slug = exToSlug.get(c.getName());
      if (slug != null) return Optional.of(slug);
      c = c.getSuperclass();
    }
    return Optional.empty();
  }

  /** Programmatic registration: add/override a slug mapping. */
  public void putSlug(String slug, Meta meta) {
    Objects.requireNonNull(slug, "slug");
    Objects.requireNonNull(meta, "meta");
    bySlug.put(slug, meta);
  }

  /** Programmatic registration: bind an exception type to a slug. */
  public void putException(Class<? extends Throwable> exceptionClass, String slug) {
    Objects.requireNonNull(exceptionClass, "exceptionClass");
    Objects.requireNonNull(slug, "slug");
    exToSlug.put(exceptionClass.getName(), slug);
  }

  /** Bulk registration helpers. */
  public void putAllSlugs(Map<String, Meta> entries) {
    if (entries != null) entries.forEach(this::putSlug);
  }
  public void putAllExceptions(Map<Class<? extends Throwable>, String> entries) {
    if (entries != null) entries.forEach(this::putException);
  }
}
```

---

## Using it with `ProblemFactory`

```java
// src/main/java/com/example/app/web/support/errors/ProblemFactory.java (excerpt)
public ProblemDetail of(String slug, String detail, Locale locale, Map<String, ?> extras) {
  var meta = requireMeta(slug);                  // ← from ExceptionMappings
  return of(meta.status(), slug, detail, locale, extras);
}
```

```java
// src/main/java/com/example/app/web/support/errors/GlobalExceptionHandler.java (excerpt)
@ExceptionHandler(UserNotFoundException.class)
ProblemDetail notFound(UserNotFoundException ex, Locale locale) {
  // If you registered ex→slug mapping:
  var slug = mappings.slugFor(ex).orElse(ProblemCatalog.RESOURCE_NOT_FOUND);
  return problems.of(slug, ex.getMessage(), locale);
}
```

---

## Feature-level extension (clean & local)

Register feature-specific mappings in a small `@Configuration` within the feature package:

```java
// src/main/java/com/example/app/web/user/support/UserExceptionMappingConfig.java
package com.example.app.web.user.support;

import com.example.app.web.support.errors.ExceptionMappings;
import com.example.app.web.support.errors.ProblemCatalog;
import org.springframework.context.annotation.Configuration;

@Configuration
class UserExceptionMappingConfig {

  UserExceptionMappingConfig(ExceptionMappings registry) {
    // Slug meta overrides or additions (rare)
    // registry.putSlug(ProblemCatalog.USER_LOCKED, new Meta(HttpStatus.LOCKED, "USER_LOCKED"));

    // Exception → slug bindings (common)
    registry.putException(UserNotFoundException.class, ProblemCatalog.RESOURCE_NOT_FOUND);
    registry.putException(EmailAlreadyUsedException.class, ProblemCatalog.VALIDATION_FAILED);
  }
}
```

This keeps the global registry central, while feature-specific knowledge stays near the feature.

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/errors/ExceptionMappingsTest.java
package com.example.app.web.support.errors;

import org.junit.jupiter.api.Test;
import org.springframework.http.HttpStatus;

import static org.assertj.core.api.Assertions.assertThat;

class ExceptionMappingsTest {

  static class MyNotFound extends RuntimeException {}
  static class MySubNotFound extends MyNotFound {}

  @Test
  void returnsMetaBySlug() {
    var m = new ExceptionMappings();
    var meta = m.meta(ProblemCatalog.RESOURCE_NOT_FOUND);
    assertThat(meta.status()).isEqualTo(HttpStatus.NOT_FOUND);
    assertThat(meta.code()).isEqualTo("NOT_FOUND");
  }

  @Test
  void registersAndResolvesExceptionToSlugWithHierarchy() {
    var m = new ExceptionMappings();
    m.putException(MyNotFound.class, ProblemCatalog.RESOURCE_NOT_FOUND);

    assertThat(m.slugFor(new MySubNotFound())).contains(ProblemCatalog.RESOURCE_NOT_FOUND);
    assertThat(m.slugFor(new IllegalStateException())).isEmpty();
  }

  @Test
  void allowsOverrideOfSlugMeta() {
    var m = new ExceptionMappings();
    m.putSlug(ProblemCatalog.VALIDATION_FAILED, new ExceptionMappings.Meta(HttpStatus.UNPROCESSABLE_ENTITY, "VALIDATION_FAILED"));
    assertThat(m.meta(ProblemCatalog.VALIDATION_FAILED).status())
        .isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY);
  }
}
```

---

## Gotchas & guardrails

* **No human strings here.** Titles belong to `ProblemText` (MessageSource). This registry is machine-only.
* **Prefer slugs over exception names.** Exception → slug is a convenience; the API contract is the `type` URI (slug).
* **Hierarchy resolution** walks superclasses to find a registered mapping—be explicit for ambiguous trees.
* **Overriding** is allowed; treat changes as API-affecting (status/code shifts can break clients).
* **Keep codes stable.** Clients often key on `code` for branching logic.

---

## Minimal checklist

* Default slug → `(status, code)` entries exist for your catalog
* Optional exception → slug bindings registered per feature
* `ProblemFactory` uses `meta(slug)` for status/code
* Tests cover slug lookups, exception hierarchy, and overrides

---

## Next helper

Hook up **`GlobalExceptionHandler`** to rely solely on `ExceptionMappings` + `ProblemFactory`, and then add **validation error shaping** (a tiny `ValidationErrors` utility) if you want consistent `errors[]` payloads.
