---
title: ProblemKey
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - i18n
    - internationalization
    - architecture
    - cheatsheet 
summary: Tiny, enum-backed problem message keys for i18n titles/details and slugs in your Spring application.
aliases:
    - Spring i18n Layer - ProblemKey Cheatsheet
---

# ProblemKey — enum-backed problem message keys

A single enum becomes the **source of truth** for:

* the **slug** used in `/problems/{slug}` and `type` URIs,
* the **i18n keys** for title/detail (`problem.<slug>.title|detail`),
* optional **default HTTP status**.

## Where it lives

* **Preferred:** `com.example.app.i18n.ProblemKey` (standalone enum)
* **Alternative:** nested under `MessageCodes` (you already did this once)

Keeping it standalone makes imports terser and avoids a huge constants class.

---

## Pattern A — minimal (slug only)

```java
// src/main/java/com/example/app/i18n/ProblemKey.java
package com.example.app.i18n;

public enum ProblemKey {
  DUPLICATE_CATEGORY("duplicate-category"),
  CATEGORY_NOT_FOUND("category-not-found"),
  INVALID_PAGINATION("invalid-pagination"),
  UNAUTHORIZED("unauthorized"),
  FORBIDDEN("forbidden"),
  CONFLICT("conflict"),
  VALIDATION_FAILED("validation-failed");

  private final String slug;
  ProblemKey(String slug) { this.slug = slug; }

  public String slug() { return slug; }

  public String titleKey()  { return "problem." + slug + ".title"; }
  public String detailKey() { return "problem." + slug + ".detail"; }
}
```

**Pros:** tiny, focused.
**Cons:** default status lives elsewhere (e.g., in `ProblemTexts` map).

---

## Pattern B — enriched (slug + default status)

```java
// src/main/java/com/example/app/i18n/ProblemKey.java
package com.example.app.i18n;

import org.springframework.http.HttpStatus;

public enum ProblemKey {
  DUPLICATE_CATEGORY("duplicate-category", HttpStatus.CONFLICT),
  CATEGORY_NOT_FOUND("category-not-found", HttpStatus.NOT_FOUND),
  INVALID_PAGINATION("invalid-pagination", HttpStatus.BAD_REQUEST),
  UNAUTHORIZED("unauthorized", HttpStatus.UNAUTHORIZED),
  FORBIDDEN("forbidden", HttpStatus.FORBIDDEN),
  CONFLICT("conflict", HttpStatus.CONFLICT),
  VALIDATION_FAILED("validation-failed", HttpStatus.UNPROCESSABLE_ENTITY);

  private final String slug;
  private final HttpStatus defaultStatus;

  ProblemKey(String slug, HttpStatus defaultStatus) {
    this.slug = slug;
    this.defaultStatus = defaultStatus;
  }

  public String slug() { return slug; }
  public HttpStatus defaultStatus() { return defaultStatus; }

  public String titleKey()  { return "problem." + slug + ".title"; }
  public String detailKey() { return "problem." + slug + ".detail"; }
}
```

**Pros:** one place to look for status + keys.
**Cons:** tiny Spring `HttpStatus` dependency in `i18n` package (usually fine).

> Pick one pattern and stick with it. If you’ve already mapped statuses inside `ProblemTexts`, Pattern A is enough.

---

## i18n bundles (titles/details)

`src/main/resources/i18n/problem-messages.properties`

```properties
problem.duplicate-category.title=Category already exists
problem.duplicate-category.detail=A category with name "{0}" already exists.

problem.category-not-found.title=Category not found
problem.category-not-found.detail=No category with id "{0}" was found.
```

---

## Wiring with your helpers

### With `Messages` + `ProblemLinks` (and `ProblemTexts` if you have it)

```java
// Build a ProblemDetail using the enum
import com.example.app.i18n.ProblemKey;
import com.example.app.i18n.Messages;
import com.example.app.web.support.links.ProblemLinks;
import org.springframework.http.ProblemDetail;

public ProblemDetail problem(Messages msg, ProblemLinks links, ProblemKey key, Object... args) {
  var pd = ProblemDetail.forStatus(
      key instanceof ProblemKey k && hasDefaultStatus ? k.defaultStatus() : org.springframework.http.HttpStatus.BAD_REQUEST
  );
  pd.setType(links.type(key.slug()));
  pd.setTitle(msg.msg(key.titleKey()));
  pd.setDetail(msg.msg(key.detailKey(), args));
  pd.setProperty("code", key.name().toLowerCase().replace('_','-'));
  return pd;
}
```

Or if you’re using the earlier `ProblemTexts` factory, just change its map to read from `key.defaultStatus()` when using Pattern B.

---

## `/problems/{slug}` docs endpoint (recap)

Your `ProblemDocController` can resolve the enum by slug:

```java
static Optional<ProblemKey> fromSlug(String slug) {
  for (var k : ProblemKey.values()) if (k.slug().equals(slug)) return Optional.of(k);
  return Optional.empty();
}
```

Then render title/detail via `Messages`, and `type` via `ProblemLinks.type(key.slug())`.

---

## How to add a new problem (one-minute SOP)

1. **Enum:** add `PAYMENT_REQUIRED("payment-required", HttpStatus.PAYMENT_REQUIRED)`.
2. **Bundles:** add:

   ```
   problem.payment-required.title=Payment required
   problem.payment-required.detail=Payment is required to access "{0}".
   ```
3. **Links:** nothing—`/problems/payment-required` just works.
4. **Throw site:** controller advice uses `ProblemKey.PAYMENT_REQUIRED` → done.

---

## Tests (quick)

```java
// src/test/java/com/example/app/i18n/ProblemKeyTest.java
package com.example.app.i18n;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ProblemKeyTest {
  @Test
  void buildsKeys() {
    var k = ProblemKey.DUPLICATE_CATEGORY;
    assertThat(k.titleKey()).isEqualTo("problem.duplicate-category.title");
    assertThat(k.detailKey()).isEqualTo("problem.duplicate-category.detail");
    assertThat(k.slug()).isEqualTo("duplicate-category");
  }
}
```

---

## Placement summary

```
src/main/java/com/example/app/
├─ i18n/
│  ├─ ProblemKey.java        # enum (Pattern A or B)
│  ├─ Messages.java
│  └─ MessageCodes.java      # optional constants, if you keep it
├─ web/support/errors/
│  ├─ ProblemTexts.java      # builder (uses ProblemKey + Messages [+ ProblemLinks])
│  └─ ProblemDocController.java
└─ web/support/links/
   └─ ProblemLinks.java
```

---

## Checklist

* `ProblemKey` enum published with stable slugs
* `problem.<slug>.title/detail` keys present in bundles
* `ProblemLinks.type(slug)` resolves canonical URIs
* `ProblemTexts` or controller advice consumes `ProblemKey`
* `/problems/{slug}` docs endpoint reflects the same enum


