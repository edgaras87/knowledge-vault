---
title: MessageCodes
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - i18n
    - internationalization
    - architecture
    - cheatsheet 
summary: Type-safe i18n keys as constants and enums to avoid stringly-typed message codes in your Spring application.
aliases:
    - Spring i18n Layer - MessageCodes Cheatsheet
---

# MessageCodes — type-safe i18n keys

Here’s a tidy, type-safe **MessageCodes** setup to kill stringly-typed keys. It pairs with your `Messages` helper and i18n bundles.

## Where it lives

* **Class:** `com.example.app.i18n.MessageCodes`

## Goals

* Centralize keys as **constants** for IDE completion.
* Provide an enum for **ProblemDetail** keys with `.titleKey()` / `.detailKey()` helpers.
* Keep structure flat and obvious; no runtime cost.

---

## Implementation

```java
// src/main/java/com/example/app/i18n/MessageCodes.java
package com.example.app.i18n;

/**
 * Central registry of message codes used across the app.
 * Keep names stable — these are part of your API contract.
 */
public final class MessageCodes {
  private MessageCodes() {}

  /** Generic app texts. */
  public static final class App {
    private App() {}
    public static final String GREETING = "app.greeting";
    public static final String ITEMS_COUNT = "app.items.count";
  }

  /** Bean Validation overrides (if you keep them separate). */
  public static final class Validation {
    private Validation() {}
    public static final String NOT_BLANK      = "javax.validation.constraints.NotBlank.message";
    public static final String SIZE_USER_NAME = "size.user.name";
    // add more as you standardize custom constraint messages
  }

  /** RFC-7807 problem keys (titles/details live in problem-messages*.properties). */
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

    /** e.g., "problem.duplicate-category.title" */
    public String titleKey()  { return "problem." + slug + ".title"; }
    /** e.g., "problem.duplicate-category.detail" */
    public String detailKey() { return "problem." + slug + ".detail"; }

    /** If you store type URIs too (optional), derive here, e.g., "/problems/duplicate-category". */
    public String typeSlug()  { return slug; }
  }
}
```

---

## Usage with `Messages`

```java
import static com.example.app.i18n.MessageCodes.App.GREETING;
import static com.example.app.i18n.MessageCodes.ProblemKey;

@Autowired Messages msg;

// Regular text
String hello = msg.msg(GREETING, "Edgaras");

// ProblemDetail texts
String title  = msg.msg(ProblemKey.DUPLICATE_CATEGORY.titleKey());
String detail = msg.msg(ProblemKey.DUPLICATE_CATEGORY.detailKey(), "Books");
```

If you use your `ProblemLinks` helper, you can align the slug via `ProblemKey.typeSlug()`.

---

## Bundle examples (for clarity)

`i18n/messages.properties`

```properties
app.greeting=Hello, {0}!
app.items.count=You have {0} item.|You have {0} items.
```

`i18n/problem-messages.properties`

```properties
problem.duplicate-category.title=Category already exists
problem.duplicate-category.detail=A category with name "{0}" already exists.

problem.category-not-found.title=Category not found
problem.category-not-found.detail=No category with id "{0}" was found.
```

---

## Tiny test

```java
// src/test/java/com/example/app/i18n/MessageCodesTest.java
package com.example.app.i18n;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class MessageCodesTest {
  @Test
  void buildsProblemKeys() {
    var k = MessageCodes.ProblemKey.DUPLICATE_CATEGORY;
    assertThat(k.titleKey()).isEqualTo("problem.duplicate-category.title");
    assertThat(k.detailKey()).isEqualTo("problem.duplicate-category.detail");
  }
}
```

---

## Tips

* Keep **enum slugs** short and URL-friendly (`duplicate-category`, `category-not-found`).
* Treat these codes as **stable contracts**—rename only with a migration (add new keys, deprecate old).
* If the list grows large, split per feature: `MessageCodesUser`, `MessageCodesCategory`, etc., and keep `MessageCodes` as a thin aggregator.


