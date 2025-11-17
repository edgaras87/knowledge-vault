---
title: ProblemText
date: 2025-11-07
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A quick reference on managing human-facing problem titles in Spring applications using MessageSource for localization and consistency in RFC-7807 problem responses.
aliases:
    - Spring Web Support Errors Layer - ProblemText Cheatsheet
---


# ProblemText — Human Titles for Problems

`ProblemText` turns a **slug** (identity) into a **title** (presentation) using Spring’s `MessageSource`.

- Keys live in bundles as `problems.<slug>.title`
- Locale-aware (accept a `Locale` from controller/advice)
- Safe default if a key is missing (no 500 because a string is absent)

Identity (`type` URI) comes from `ProblemLinks`.  
Presentation (`title`) comes from `ProblemText`.

---

## Where it lives

- **Class:** `com.example.app.web.support.errors.ProblemText`
- **Depends on:** Spring `MessageSource`
- **Used by:** `GlobalExceptionHandler`, `ProblemFactory`, tests

---

## Resource bundles

Create message files under `src/main/resources/i18n/`:

```
src/main/resources/i18n/
problem-messages.properties
problem-messages_lt.properties     # example extra locale
```

**Keys (example)** — `src/main/resources/i18n/problem-messages.properties`

```properties
problems.resource-not-found.title=Resource not found
problems.validation-failed.title=Validation failed
problems.malformed-request.title=Malformed request
problems.internal-error.title=Internal error
```

**Spring config** — declare basenames (one or many):

```yaml
spring:
  messages:
    basename: i18n/problem-messages
    fallback-to-system-locale: false   # prefer Accept-Language/explicit Locale
```

*(If you already have a global `messages` bundle, you can include multiple basenames separated by commas.)*

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/errors/ProblemText.java
package com.example.app.web.support.errors;

import org.springframework.context.MessageSource;
import org.springframework.context.NoSuchMessageException;
import org.springframework.stereotype.Component;

import java.util.Locale;
import java.util.Objects;

@Component
public class ProblemText {

  private static final String KEY_PREFIX = "problems.";
  private static final String KEY_SUFFIX_TITLE = ".title";

  private final MessageSource messages;

  public ProblemText(MessageSource messages) {
    this.messages = messages;
  }

  /** Title for a given slug, localized by provided locale (must not be null). */
  public String title(String slug, Locale locale) {
    Objects.requireNonNull(locale, "locale");
    var key = keyForTitle(slug);
    try {
      return messages.getMessage(key, null, locale);
    } catch (NoSuchMessageException e) {
      // Safe fallback to avoid 500s if bundle is missing a key.
      return fallbackTitle(slug);
    }
  }

  // -------- helpers (package-private for tests) --------

  static String keyForTitle(String slug) {
    validateSlug(slug);
    return KEY_PREFIX + slug + KEY_SUFFIX_TITLE;
  }

  static void validateSlug(String slug) {
    if (slug == null || slug.isBlank())
      throw new IllegalArgumentException("Problem slug must be non-empty");
    if (!slug.matches("[a-z0-9\\-\\.]+"))
      throw new IllegalArgumentException("Problem slug must match [a-z0-9-.]+");
  }

  static String fallbackTitle(String slug) {
    // naive humanization: "resource-not-found" -> "Resource not found"
    var pretty = slug.replace('-', ' ').replace('.', ' ');
    return Character.toUpperCase(pretty.charAt(0)) + pretty.substring(1);
  }
}
```

---

## Usage in `@ControllerAdvice` (excerpt)

```java
import static com.example.app.web.support.errors.ProblemCatalog.*;

@ControllerAdvice
public class GlobalExceptionHandler {

  private final ProblemLinks links;
  private final ProblemText text;

  public GlobalExceptionHandler(ProblemLinks links, ProblemText text) {
    this.links = links;
    this.text = text;
  }

  @ExceptionHandler(ResourceNotFoundException.class)
  ProblemDetail notFound(ResourceNotFoundException ex, Locale locale) {
    var slug = RESOURCE_NOT_FOUND;
    var pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
    pd.setType(links.type(slug));
    pd.setTitle(text.title(slug, locale));   // ← localized, safe
    pd.setDetail(ex.getMessage());
    pd.setProperty("code", "NOT_FOUND");
    return pd;
  }
}
```

> Inject `Locale` as a method parameter; Spring resolves it from `Accept-Language` or your `LocaleResolver`.

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/errors/ProblemTextTest.java
package com.example.app.web.support.errors;

import org.junit.jupiter.api.Test;
import org.springframework.context.support.StaticMessageSource;

import java.util.Locale;

import static org.assertj.core.api.Assertions.*;

class ProblemTextTest {

  @Test
  void resolvesFromMessageSourceOrFallsBack() {
    var ms = new StaticMessageSource();
    ms.addMessage("problems.resource-not-found.title", Locale.ENGLISH, "Resource not found");
    var pt = new ProblemText(ms);

    assertThat(pt.title("resource-not-found", Locale.ENGLISH)).isEqualTo("Resource not found");
    assertThat(pt.title("validation-failed", Locale.ENGLISH)).isEqualTo("Validation failed"); // fallback humanization
  }

  @Test
  void validatesSlug() {
    assertThatThrownBy(() -> ProblemText.keyForTitle(" NotOk "))
        .isInstanceOf(IllegalArgumentException.class);
  }
}
```

---

## Gotchas & guardrails

* **Don’t** store titles in code. Keep them in bundles; slugs stay in code.
* **Do** provide a fallback to avoid error responses failing due to missing keys.
* **Keep keys stable**: changing keys breaks translations; changing slugs breaks clients.
* **Locale**: pass it explicitly in handlers; avoid static/global locale lookups.
* **Basenames**: ensure `spring.messages.basename` includes `i18n/problem-messages` (and others if you split bundles).

---

## Minimal checklist

* `problem-messages.properties` exists with `problems.<slug>.title` keys
* `ProblemText.title(slug, locale)` returns a string for all known slugs
* Fallback humanization works if a key is missing
* Tests cover resolution + fallback + slug validation

---

## Next helper

Add **`ProblemFactory`** to DRY creation of `ProblemDetail` (status + slug + title + optional detail), then wire **`ExceptionMappings`** to centralize machine codes and statuses.


