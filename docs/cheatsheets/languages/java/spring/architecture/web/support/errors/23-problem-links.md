---
title: ProblemLinks
date: 2025-11-07
tags: 
    - java
    - spring
    - web
    - architecture
    - support
    - cheatsheet
summary: A cheatsheet for implementing stable ProblemDetail type URIs in Java Spring applications using a centralized ProblemLinks helper.
aliases:
    - Spring Web Support Errors Layer - ProblemLinks Cheatsheet
---



# ProblemLinks — Stable Problem `type` URIs

`ProblemLinks` converts a **slug** (e.g., `resource-not-found`) into a canonical, absolute `type` URI:

```
[https://example.com/problems/resource-not-found](https://example.com/problems/resource-not-found)
```

- Uses `UriFactory` to get a normalized **host** and a `/problems/` **base**.
- **Not versioned**: problem identities should be stable over time.
- Pure utility for the web edge—ideal for `@ControllerAdvice` and tests.

---

## Where it lives

- **Class:** `com.example.app.web.support.errors.ProblemLinks`
- **Depends on:** `com.example.app.web.support.links.UriFactory`
- **Used by:** `GlobalExceptionHandler`, `ProblemFactory`, tests

---

## Inputs

Problem slugs are defined once (e.g., a tiny catalog):

```java
// src/main/java/com/example/app/web/support/errors/ProblemCatalog.java
package com.example.app.web.support.errors;

public final class ProblemCatalog {
  private ProblemCatalog() {}
  public static final String RESOURCE_NOT_FOUND = "resource-not-found";
  public static final String VALIDATION_FAILED  = "validation-failed";
  public static final String MALFORMED_REQUEST  = "malformed-request";
  public static final String INTERNAL_ERROR     = "internal-error";
}
```

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/errors/ProblemLinks.java
package com.example.app.web.support.errors;

import com.example.app.web.support.links.UriFactory;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.util.Objects;

@Component
public class ProblemLinks {

  private final UriFactory uriFactory;
  private final URI problemsBase; // e.g., https://example.com/problems/

  public ProblemLinks(UriFactory uriFactory) {
    this.uriFactory = uriFactory;
    this.problemsBase = uriFactory.baseUnderHost("problems");
  }

  /** The normalized problems base. Always ends with '/'. */
  public URI base() {
    return problemsBase;
  }

  /**
   * Resolve a problem 'type' URI for the given slug.
   * Example: "resource-not-found" → https://example.com/problems/resource-not-found
   */
  public URI type(String slug) {
    validateSlug(slug);
    return uriFactory.resolve(problemsBase, slug);
  }

  // ---------- helpers (package-private for tests) ----------

  static void validateSlug(String slug) {
    if (slug == null || slug.isBlank()) {
      throw new IllegalArgumentException("Problem slug must be non-empty");
    }
    // minimal sanity: lowercase letters, digits, dash; adjust if you need dots or underscores
    if (!slug.matches("[a-z0-9\\-\\.]+")) {
      throw new IllegalArgumentException("Problem slug must match [a-z0-9-.]+");
    }
  }
}
```

---

## Usage in a global exception handler

```java
// src/main/java/com/example/app/web/support/errors/GlobalExceptionHandler.java (excerpt)
import static com.example.app.web.support.errors.ProblemCatalog.*;

@ControllerAdvice
public class GlobalExceptionHandler {

  private final ProblemLinks links;

  public GlobalExceptionHandler(ProblemLinks links) { this.links = links; }

  @ExceptionHandler(ResourceNotFoundException.class)
  ProblemDetail notFound(ResourceNotFoundException ex) {
    var pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
    pd.setType(links.type(RESOURCE_NOT_FOUND));   // ← stable RFC-7807 type
    pd.setTitle("Resource not found");            // (ProblemText will replace this later)
    pd.setDetail(ex.getMessage());
    pd.setProperty("code", "NOT_FOUND");
    return pd;
  }
}
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/errors/ProblemLinksTest.java
package com.example.app.web.support.errors;

import com.example.app.config.AppProps;
import com.example.app.web.support.links.UriFactory;
import org.junit.jupiter.api.Test;

import java.net.URI;

import static org.assertj.core.api.Assertions.*;

class ProblemLinksTest {

  @Test
  void buildsBaseAndResolvesSlug() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var factory = new UriFactory(props);
    var links = new ProblemLinks(factory);

    assertThat(links.base().toString()).isEqualTo("https://example.com/problems/");
    assertThat(links.type("resource-not-found").toString())
        .isEqualTo("https://example.com/problems/resource-not-found");
  }

  @Test
  void rejectsBadSlug() {
    var links = new ProblemLinks(new UriFactory(new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"))));
    assertThatThrownBy(() -> links.type(" Not Ok "))
        .isInstanceOf(IllegalArgumentException.class);
  }
}
```

---

## Gotchas & guardrails

* **Don’t** version `/problems/` by default. If you must version, prefer versioning the **slug** (`validation-failed.v2`) or make a separate versioned base (another factory method).
* **Do** keep slugs stable; changing a slug is a breaking change for clients keying on `type`.
* **Don’t** compute host from request; rely on `UriFactory` so tests are deterministic.
* **Do** centralize slugs in `ProblemCatalog` to avoid typos.

---

## Minimal checklist

* `base()` returns `https://example.com/problems/`
* `type(slug)` validates and resolves to `https://example.com/problems/{slug}`
* Exception handler sets `ProblemDetail.type` via `ProblemLinks`
* Unit tests cover base + slug resolution and invalid slugs

---

## Next helper

Wire **`ProblemText`** to resolve human titles (`problems.<slug>.title`) from `MessageSource`, then optionally **`ProblemFactory`** to DRY creation of `ProblemDetail`.
