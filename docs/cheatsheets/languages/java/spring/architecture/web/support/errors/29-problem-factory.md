---
title: ProblemFactory 
date: 2025-11-08
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A quick reference on creating RFC-7807 ProblemDetail instances in Spring applications using a centralized ProblemFactory for consistency and maintainability.
aliases:
    - Spring Web Support Errors Layer - ProblemFactory Cheatsheet
---

# ProblemFactory — One-Liner ProblemDetail

`ProblemFactory` centralizes the boring bits of RFC-7807 construction:

- `type` from **ProblemLinks** (slug → absolute URI)
- `title` from **ProblemText** (slug → localized title)
- `status` from **HttpStatus** (or from **ExceptionMappings** if you use it)
- optional `detail` and machine `code`
- optional extras (e.g., `traceId`, `errors[]`)

Controllers and advices stop repeating themselves and stay uniform.

---

## Where it lives

- **Class:** `com.example.app.web.support.errors.ProblemFactory`
- **Depends on:** `ProblemLinks`, `ProblemText` (optionally `ExceptionMappings`)
- **Used by:** `GlobalExceptionHandler`, feature translators, tests

---

## Minimal API (you choose one)

1) **Explicit status each time**  
   `of(status, slug, detail, locale)`  
2) **Through a registry** (pair with `ExceptionMappings`)  
   `of(slug, detail, locale)` → status & code come from mapping

Both are implemented below.

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/errors/ProblemFactory.java
package com.example.app.web.support.errors;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.util.Locale;
import java.util.Map;
import java.util.Objects;

@Component
public class ProblemFactory {

  private final ProblemLinks links;
  private final ProblemText text;
  private final ExceptionMappings mappings; // optional; can be a no-op if you don't wire it

  public ProblemFactory(ProblemLinks links, ProblemText text, ExceptionMappings mappings) {
    this.links = links;
    this.text = text;
    this.mappings = mappings;
  }

  /** Build a ProblemDetail with explicit status. Adds stable type/title and optional code/detail/extras. */
  public ProblemDetail of(HttpStatus status, String slug, String detail, Locale locale, Map<String, ?> extras) {
    Objects.requireNonNull(status, "status");
    Objects.requireNonNull(slug, "slug");
    Objects.requireNonNull(locale, "locale");

    URI type = links.type(slug);
    String title = text.title(slug, locale);

    ProblemDetail pd = ProblemDetail.forStatus(status);
    pd.setType(type);
    pd.setTitle(title);
    if (detail != null && !detail.isBlank()) pd.setDetail(detail);

    // If you have a registry entry, include machine code by default
    var meta = mappings.meta(slug); // may be null
    if (meta != null && meta.code() != null) {
      pd.setProperty("code", meta.code());
    }

    if (extras != null && !extras.isEmpty()) {
      extras.forEach(pd::setProperty);
    }
    return pd;
  }

  /** Convenience: use ExceptionMappings for status/code; you only provide slug, detail, locale, extras. */
  public ProblemDetail of(String slug, String detail, Locale locale, Map<String, ?> extras) {
    var meta = requireMeta(slug);
    return of(meta.status(), slug, detail, locale, extras);
  }

  // ----- convenience overloads -----

  public ProblemDetail of(HttpStatus status, String slug, String detail, Locale locale) {
    return of(status, slug, detail, locale, Map.of());
  }

  public ProblemDetail of(String slug, String detail, Locale locale) {
    return of(slug, detail, locale, Map.of());
  }

  // ----- helpers -----

  private ExceptionMappings.Meta requireMeta(String slug) {
    var meta = mappings.meta(slug);
    if (meta == null) throw new IllegalArgumentException("No ExceptionMappings meta for slug: " + slug);
    return meta;
  }
}
```

---

## Optional: ExceptionMappings (machine registry)

If you haven’t created it yet, here’s a compact version:

```java
// src/main/java/com/example/app/web/support/errors/ExceptionMappings.java
package com.example.app.web.support.errors;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class ExceptionMappings {

  public record Meta(HttpStatus status, String code) {}

  // Adjust to your catalog
  private static final Map<String, Meta> META = Map.of(
    ProblemCatalog.RESOURCE_NOT_FOUND, new Meta(HttpStatus.NOT_FOUND, "NOT_FOUND"),
    ProblemCatalog.VALIDATION_FAILED,  new Meta(HttpStatus.BAD_REQUEST, "VALIDATION_FAILED"),
    ProblemCatalog.INTERNAL_ERROR,     new Meta(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERROR")
  );

  public Meta meta(String slug) { return META.get(slug); }
}
```

---

## Usage in `@ControllerAdvice` (concise)

```java
import static com.example.app.web.support.errors.ProblemCatalog.*;

@ControllerAdvice
public class GlobalExceptionHandler {

  private final ProblemFactory problems;

  public GlobalExceptionHandler(ProblemFactory problems) {
    this.problems = problems;
  }

  @ExceptionHandler(ResourceNotFoundException.class)
  ProblemDetail notFound(ResourceNotFoundException ex, Locale locale) {
    // Using registry-based overload: status & code come from ExceptionMappings
    return problems.of(RESOURCE_NOT_FOUND, ex.getMessage(), locale);
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  ProblemDetail validation(MethodArgumentNotValidException ex, Locale locale) {
    var errors = ValidationErrors.from(ex); // your own list of field errors
    return problems.of(
        ExceptionMappingsMetaOr(HttpStatus.BAD_REQUEST), // or just problems.of(VALIDATION_FAILED, ...)
        ProblemCatalog.VALIDATION_FAILED,
        "Request validation failed",
        locale,
        Map.of("errors", errors));
  }

  @ExceptionHandler(Exception.class)
  ProblemDetail catchAll(Exception ex, Locale locale) {
    // Prefer neutral detail; code comes from mappings
    return problems.of(ProblemCatalog.INTERNAL_ERROR, "An unexpected error occurred.", locale);
  }

  private HttpStatus ExceptionMappingsMetaOr(HttpStatus fallback) { return fallback; } // placeholder for demo
}
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/errors/ProblemFactoryTest.java
package com.example.app.web.support.errors;

import com.example.app.config.AppProps;
import com.example.app.web.support.links.UriFactory;
import org.junit.jupiter.api.Test;
import org.springframework.context.support.StaticMessageSource;
import org.springframework.http.HttpStatus;

import java.net.URI;
import java.util.Locale;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ProblemFactoryTest {

  @Test
  void buildsProblemWithTypeTitleStatusCodeAndExtras() {
    // infra
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var uriFactory = new UriFactory(props);
    var links = new ProblemLinks(uriFactory);

    var ms = new StaticMessageSource();
    ms.addMessage("problems.resource-not-found.title", Locale.ENGLISH, "Resource not found");
    var text = new ProblemText(ms);

    var mappings = new ExceptionMappings();
    var factory = new ProblemFactory(links, text, mappings);

    var pd = factory.of(HttpStatus.NOT_FOUND, ProblemCatalog.RESOURCE_NOT_FOUND, "User 42 not found", Locale.ENGLISH,
        Map.of("traceId", "abc-123"));

    assertThat(pd.getStatus()).isEqualTo(404);
    assertThat(pd.getTitle()).isEqualTo("Resource not found");
    assertThat(pd.getType().toString()).isEqualTo("https://example.com/problems/resource-not-found");
    assertThat(pd.getDetail()).isEqualTo("User 42 not found");
    assertThat(pd.getProperties()).containsEntry("code", "NOT_FOUND")
                                  .containsEntry("traceId", "abc-123");
  }

  @Test
  void registryOverloadPicksStatusFromMappings() {
    var factory = new ProblemFactory(
        new ProblemLinks(new UriFactory(new AppProps(URI.create("https://example.com"), new AppProps.Api("v1")))),
        new ProblemText(new StaticMessageSource()),
        new ExceptionMappings()
    );

    var pd = factory.of(ProblemCatalog.INTERNAL_ERROR, "boom", Locale.ENGLISH);
    assertThat(pd.getStatus()).isEqualTo(500);
  }
}
```

---

## Gotchas & guardrails

* **Slug is the key**: identity is stable; titles may change per locale. Keep slugs in `ProblemCatalog`.
* **One place for machine fields**: status + `code` live in `ExceptionMappings`. Don’t sprinkle magic strings in handlers.
* **Safe defaults**: `ProblemText` must never cause 500s—fallback if a key is missing.
* **Extras are optional**: attach `traceId`, `errors[]`, `instance` if you have it; keep the shape predictable.
* **No servlet context**: everything here is pure and unit-testable.

---

## Minimal checklist

* `of(status, slug, detail, locale[, extras])` exists
* Registry-based `of(slug, detail, locale[, extras])` exists (if you use `ExceptionMappings`)
* Sets `type`, `title`, `status`, and `code` (when mapped)
* Accepts `extras` map and merges into `properties`
* Tests cover both overloads and property assertions

---

## Next helper

Finalize the machine registry with **`ExceptionMappings`** (if you haven’t already), then wire your **`GlobalExceptionHandler`** to only call `ProblemFactory`—no more hand-built `ProblemDetail`s scattered across the codebase.











