---
title: ProblemText (+i18n)
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
    - Spring Web Support Errors Layer - ProblemText i18n Cheatsheet
---



# ProblemTexts — RFC-7807 builder (i18n + typed links)

## Where it lives

* **Class:** `com.example.app.web.support.errors.ProblemTexts`
* **Depends on:** `com.example.app.i18n.Messages`, `com.example.app.web.support.links.ProblemLinks`, `com.example.app.i18n.MessageCodes.ProblemKey`

> Assumes you already have `ProblemLinks` with `URI type(String slug)` (or `base()` + `type(...)`) and your `Messages` + `MessageCodes`.

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/errors/ProblemTexts.java
package com.example.app.web.support.errors;

import com.example.app.i18n.MessageCodes.ProblemKey;
import com.example.app.i18n.Messages;
import com.example.app.web.support.links.ProblemLinks;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.util.EnumMap;
import java.util.Map;
import java.util.Objects;

/**
 * Factory for RFC-7807 ProblemDetail with i18n title/detail and canonical type URI.
 */
@Component
public class ProblemTexts {

  private final Messages msg;
  private final ProblemLinks links;
  private final Map<ProblemKey, HttpStatus> defaults = new EnumMap<>(ProblemKey.class);

  public ProblemTexts(Messages msg, ProblemLinks links) {
    this.msg = Objects.requireNonNull(msg);
    this.links = Objects.requireNonNull(links);

    // Sensible defaults; extend as you add keys
    defaults.put(ProblemKey.DUPLICATE_CATEGORY, HttpStatus.CONFLICT);        // 409
    defaults.put(ProblemKey.CATEGORY_NOT_FOUND, HttpStatus.NOT_FOUND);       // 404
    defaults.put(ProblemKey.INVALID_PAGINATION, HttpStatus.BAD_REQUEST);     // 400
    defaults.put(ProblemKey.UNAUTHORIZED,       HttpStatus.UNAUTHORIZED);    // 401
    defaults.put(ProblemKey.FORBIDDEN,          HttpStatus.FORBIDDEN);       // 403
    defaults.put(ProblemKey.CONFLICT,           HttpStatus.CONFLICT);        // 409
    defaults.put(ProblemKey.VALIDATION_FAILED,  HttpStatus.UNPROCESSABLE_ENTITY); // 422
  }

  /** Build with the default status for the key. */
  public ProblemDetail of(ProblemKey key, Object... args) {
    return of(defaults.getOrDefault(key, HttpStatus.BAD_REQUEST), key, args);
  }

  /** Build with an explicit status, overriding the default. */
  public ProblemDetail of(HttpStatus status, ProblemKey key, Object... args) {
    var pd = ProblemDetail.forStatus(status);
    pd.setType(typeUri(key));
    pd.setTitle(msg.msg(key.titleKey()));
    pd.setDetail(msg.msg(key.detailKey(), args));
    // Optional: add extension members here (e.g., a stable error code)
    pd.setProperty("code", key.name().toLowerCase().replace('_','-'));
    return pd;
  }

  private URI typeUri(ProblemKey key) {
    return links.type(key.typeSlug()); // e.g., /problems/duplicate-category
  }
}
```

---

## Minimal `ProblemLinks` (reference)

If you don’t already have it or want a lean version:

```java
// src/main/java/com/example/app/web/support/links/ProblemLinks.java
package com.example.app.web.support.links;

import com.example.app.config.AppProps;
import org.springframework.stereotype.Component;

import java.net.URI;

@Component
public class ProblemLinks {
  private final URI base;

  public ProblemLinks(AppProps props) {
    // normalize once; ensure trailing slash
    var host = props.host().toString();
    this.base = URI.create(host.endsWith("/") ? host : host + "/");
  }

  public URI base() { return base.resolve("problems/"); }
  public URI type(String slug) { return base().resolve(slug); } // "/problems/{slug}"
}
```

---

## Usage in a controller advice

```java
// src/main/java/com/example/app/web/support/errors/ApiExceptionHandler.java
package com.example.app.web.support.errors;

import com.example.app.i18n.MessageCodes.ProblemKey;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class ApiExceptionHandler {

  private final ProblemTexts problems;

  public ApiExceptionHandler(ProblemTexts problems) {
    this.problems = problems;
  }

  @ExceptionHandler(DuplicateCategoryException.class)
  public ResponseEntity<?> dupCategory(DuplicateCategoryException ex) {
    var pd = problems.of(ProblemKey.DUPLICATE_CATEGORY, ex.getName());
    return ResponseEntity.status(pd.getStatus()).body(pd);
  }

  @ExceptionHandler(CategoryNotFoundException.class)
  public ResponseEntity<?> notFound(CategoryNotFoundException ex) {
    var pd = problems.of(ProblemKey.CATEGORY_NOT_FOUND, ex.getId());
    return ResponseEntity.status(pd.getStatus()).body(pd);
  }
}
```

---

## Bundle keys (recap)

`i18n/problem-messages.properties`

```properties
problem.duplicate-category.title=Category already exists
problem.duplicate-category.detail=A category with name "{0}" already exists.

problem.category-not-found.title=Category not found
problem.category-not-found.detail=No category with id "{0}" was found.
```

---

## Quick test sketch

```java
// src/test/java/com/example/app/web/support/errors/ProblemTextsTest.java
package com.example.app.web.support.errors;

import com.example.app.i18n.MessageCodes.ProblemKey;
import com.example.app.i18n.Messages;
import org.junit.jupiter.api.Test;
import org.springframework.context.support.StaticMessageSource;

import java.net.URI;
import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

class ProblemTextsTest {

  @Test
  void buildsProblemDetail() {
    var sms = new StaticMessageSource();
    sms.addMessage("problem.duplicate-category.title", Locale.ENGLISH, "Category already exists");
    sms.addMessage("problem.duplicate-category.detail", Locale.ENGLISH, "Name \"{0}\" exists");

    var messages = new Messages(sms);

    var links = new ProblemLinksStub(URI.create("https://api.example.com/"));
    var texts = new ProblemTexts(messages, links);

    var pd = texts.of(ProblemKey.DUPLICATE_CATEGORY, "Books");
    assertThat(pd.getTitle()).contains("exists");
    assertThat(pd.getType().toString()).isEqualTo("https://api.example.com/problems/duplicate-category");
  }

  static final class ProblemLinksStub extends com.example.app.web.support.links.ProblemLinks {
    private final URI base;
    ProblemLinksStub(URI host) { super(new DummyProps(host)); this.base = host; }
    @Override public URI base() { return base.resolve("problems/"); }
  }
  // DummyProps: minimal stand-in if needed (or just use a direct stub implementation).
}
```

---

## Notes

* **Status mapping** lives inside `ProblemTexts` to keep controllers skinny. If you prefer, move it to an external map or properties.
* The `code` extension member gives clients a stable machine key independent of localized title/detail.
* You can add more extensions (e.g., `instance`, `traceId`) right before returning the response.
