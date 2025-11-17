---
title: ProblemCatalog 
date: 2025-11-08
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A quick reference on managing canonical slugs for RFC-7807 problems in Spring applications using a centralized ProblemCatalog for consistent problem identity across type URIs, i18n keys, and exception mappings.
aliases:
    - Spring Web Support Errors Layer - ProblemCatalog Cheatsheet
---

# ProblemCatalog — Canonical Slugs

`ProblemCatalog` is a tiny class that defines **stable slugs** (string constants) for every RFC-7807 problem your API emits.  
Slugs are the **identity** behind:

- `type` URIs → `https://example.com/problems/<slug>`
- i18n keys → `problems.<slug>.title`
- machine registry → `ExceptionMappings` (status + `code`)

Keep them in one place; treat changes as breaking.

---

## Where it lives

- **Class:** `com.example.app.web.support.errors.ProblemCatalog`
- **Used by:** `ProblemLinks`, `ProblemText`, `ProblemFactory`, `ExceptionMappings`, `@ControllerAdvice` handlers, tests
- **Not** an enum — plain `public static final String` constants (easy to use in annotations, properties, DB seeds)

---

## Implementation (copy-paste)

```java
// src/main/java/com/example/app/web/support/errors/ProblemCatalog.java
package com.example.app.web.support.errors;

/**
 * Canonical slugs for RFC-7807 problems.
 * Keep these STABLE. Changing a slug is a breaking change for clients.
 *
 * Conventions:
 *  - kebab-case
 *  - lowercase letters, digits, dashes ('.' allowed only if you deliberately version a slug, e.g., validation-failed.v2)
 */
public final class ProblemCatalog {
  private ProblemCatalog() {}

  // --- 4xx (client errors) ---
  public static final String VALIDATION_FAILED    = "validation-failed";
  public static final String MALFORMED_REQUEST    = "malformed-request";    // unreadable/invalid JSON, type mismatches
  public static final String RESOURCE_NOT_FOUND   = "resource-not-found";
  public static final String UNAUTHORIZED         = "unauthorized";
  public static final String FORBIDDEN            = "forbidden";
  public static final String CONFLICT             = "conflict";
  public static final String RATE_LIMITED         = "rate-limited";

  // --- 5xx (server errors) ---
  public static final String INTERNAL_ERROR       = "internal-error";
  public static final String SERVICE_UNAVAILABLE  = "service-unavailable";

  // --- Optional: feature-specific slugs (group logically) ---
  // public static final String USER_LOCKED       = "user-locked";
  // public static final String EMAIL_ALREADY_USED= "email-already-used";
}
```

---

## i18n keys (align with catalog)

Create `src/main/resources/i18n/problem-messages.properties`:

```properties
problems.validation-failed.title=Validation failed
problems.malformed-request.title=Malformed request
problems.resource-not-found.title=Resource not found
problems.unauthorized.title=Unauthorized
problems.forbidden.title=Forbidden
problems.conflict.title=Conflict
problems.rate-limited.title=Too many requests
problems.internal-error.title=Internal error
problems.service-unavailable.title=Service temporarily unavailable
```

And register the basename:

```yaml
spring:
  messages:
    basename: i18n/problem-messages
```

---

## Machine registry (align with catalog)

In `ExceptionMappings`:

```java
putSlug(ProblemCatalog.VALIDATION_FAILED,   new Meta(HttpStatus.BAD_REQUEST, "VALIDATION_FAILED"));
putSlug(ProblemCatalog.MALFORMED_REQUEST,   new Meta(HttpStatus.BAD_REQUEST, "MALFORMED_REQUEST"));
putSlug(ProblemCatalog.RESOURCE_NOT_FOUND,  new Meta(HttpStatus.NOT_FOUND,   "NOT_FOUND"));
putSlug(ProblemCatalog.UNAUTHORIZED,        new Meta(HttpStatus.UNAUTHORIZED,"UNAUTHORIZED"));
putSlug(ProblemCatalog.FORBIDDEN,           new Meta(HttpStatus.FORBIDDEN,   "FORBIDDEN"));
putSlug(ProblemCatalog.CONFLICT,            new Meta(HttpStatus.CONFLICT,    "CONFLICT"));
putSlug(ProblemCatalog.RATE_LIMITED,        new Meta(HttpStatus.TOO_MANY_REQUESTS, "RATE_LIMITED"));
putSlug(ProblemCatalog.INTERNAL_ERROR,      new Meta(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"));
putSlug(ProblemCatalog.SERVICE_UNAVAILABLE, new Meta(HttpStatus.SERVICE_UNAVAILABLE, "SERVICE_UNAVAILABLE"));
```

---

## Usage examples

```java
// GlobalExceptionHandler
@ExceptionHandler(ResourceNotFoundException.class)
ProblemDetail notFound(ResourceNotFoundException ex, Locale locale) {
  return problems.of(ProblemCatalog.RESOURCE_NOT_FOUND, ex.getMessage(), locale);
}

// ProblemLinks
URI type = problemLinks.type(ProblemCatalog.VALIDATION_FAILED);

// ProblemText
String title = problemText.title(ProblemCatalog.INTERNAL_ERROR, locale);
```

---

## Tests (sanity)

```java
// src/test/java/com/example/app/web/support/errors/ProblemCatalogTest.java
package com.example.app.web.support.errors;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ProblemCatalogTest {

  @Test
  void slugsAreKebabCaseAndNonEmpty() {
    String[] slugs = {
        ProblemCatalog.VALIDATION_FAILED,
        ProblemCatalog.MALFORMED_REQUEST,
        ProblemCatalog.RESOURCE_NOT_FOUND,
        ProblemCatalog.UNAUTHORIZED,
        ProblemCatalog.FORBIDDEN,
        ProblemCatalog.CONFLICT,
        ProblemCatalog.RATE_LIMITED,
        ProblemCatalog.INTERNAL_ERROR,
        ProblemCatalog.SERVICE_UNAVAILABLE
    };
    for (var s : slugs) {
      assertThat(s).isNotBlank();
      assertThat(s).matches("[a-z0-9\\-\\.]+");
    }
  }
}
```

---

## Guardrails

* **Stable forever:** changes to a slug are **breaking** (clients often link docs by `type`).
* **One place only:** add new slugs here first, then wire i18n and mappings.
* **Naming:** prefer concise, action-oriented names (`validation-failed`, not `there-were-some-validation-problems`).
* **Versioning (rare):** if you must, embed in slug (`validation-failed.v2`) **or** create a versioned problems base (separate policy).

---

## Checklist

* Central `ProblemCatalog` exists with kebab-case slugs
* Message bundles define `problems.<slug>.title`
* `ExceptionMappings` defines `(status, code)` for each slug
* `ProblemLinks` resolves `type` URIs using these slugs
* Tests ensure slugs meet the allowed pattern
