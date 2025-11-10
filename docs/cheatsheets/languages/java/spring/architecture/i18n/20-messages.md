---
title: Messages
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - i18n
    - internationalization
    - architecture
    - cheatsheet
summary: Tiny, ergonomic helper for i18n message resolution in Spring applications. A friendly wrapper around MessageSource that respects the request locale and offers convenient methods for fetching localized strings, including RFC-7807 problem titles and details.
aliases:
    - Spring i18n Layer - Messages Cheatsheet
---


---

# Messages — a friendly wrapper around `MessageSource`

Here’s a tiny, ergonomic `Messages` helper that wraps Spring’s `MessageSource` so you can write `msg("problem.duplicate-category.detail", name)` anywhere without ceremony. It respects the request locale via `LocaleContextHolder`, but you can also target a specific `Locale`.

## Where it lives

* **Class:** `com.example.app.i18n.Messages`
* **Config (bean):** `com.example.app.config.MessagesConfig`
* **Used by:** controllers, exception handlers, services (e.g., building RFC-7807 titles/details, validation summaries, e-mail text)

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/i18n/Messages.java
package com.example.app.i18n;

import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;

import java.util.Locale;
import java.util.Objects;
import java.util.Optional;

/**
 * Tiny facade over MessageSource with sane defaults.
 * - Uses LocaleContextHolder locale by default (from WebConfig's resolver).
 * - Provides tryGet / getOrDefault ergonomics.
 */
public final class Messages {
  private final MessageSource ms;

  public Messages(MessageSource ms) {
    this.ms = Objects.requireNonNull(ms, "messageSource");
  }

  /** Resolve with current request locale (LocaleContextHolder). */
  public String msg(String code, Object... args) {
    Locale locale = LocaleContextHolder.getLocale();
    return ms.getMessage(code, args, code, locale);
  }

  /** Resolve with explicit locale. */
  public String msg(Locale locale, String code, Object... args) {
    return ms.getMessage(code, args, code, locale);
  }

  /** Try to resolve; empty if not found (never throws). */
  public Optional<String> tryMsg(String code, Object... args) {
    Locale locale = LocaleContextHolder.getLocale();
    String value = ms.getMessage(code, args, null, locale);
    return Optional.ofNullable(value);
  }

  /** Resolve or return a provided fallback (not the code). */
  public String msgOrDefault(String code, String fallback, Object... args) {
    Locale locale = LocaleContextHolder.getLocale();
    return ms.getMessage(code, args, fallback, locale);
  }

  // ---------- RFC-7807 convenience (optional but handy) ----------

  /** problem.{key}.title  (e.g., "problem.duplicate-category.title") */
  public String problemTitle(String key, Object... args) {
    return msg("problem." + key + ".title", args);
  }

  /** problem.{key}.detail (e.g., "problem.duplicate-category.detail") */
  public String problemDetail(String key, Object... args) {
    return msg("problem." + key + ".detail", args);
  }
}
```

### Wire it as a bean

```java
// src/main/java/com/example/app/config/MessagesConfig.java
package com.example.app.config;

import com.example.app.i18n.Messages;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MessagesConfig {

  @Bean
  public Messages messages(MessageSource messageSource) {
    return new Messages(messageSource);
  }
}
```

> This reuses your existing `MessageSourceConfig` bean and your `WebConfig` locale resolver. No duplication.

---

## Usage examples

### 1) ProblemDetail handler (controller advice)

```java
// src/main/java/com/example/app/web/support/errors/ProblemTexts.java
package com.example.app.web.support.errors;

import com.example.app.i18n.Messages;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;

@Component
public class ProblemTexts {
  private final Messages msg;

  public ProblemTexts(Messages msg) { this.msg = msg; }

  public ProblemDetail duplicateCategory(String name) {
    var pd = ProblemDetail.forStatus(409);
    pd.setTitle(msg.problemTitle("duplicate-category"));     // i18n/problem-messages.properties
    pd.setDetail(msg.problemDetail("duplicate-category", name));
    return pd;
  }
}
```

### 2) Service or controller

```java
// in any @Service / @RestController
@Autowired Messages msg;

// ...
String greeting = msg.msg("app.greeting", "Edgaras"); // → "Hello, Edgaras!"
```

### 3) Explicit locale (rare, but sometimes needed)

```java
var de = java.util.Locale.GERMAN;
String titleDe = msg.msg(de, "problem.duplicate-category.title");
```

---

## Message bundle examples (for completeness)

`src/main/resources/i18n/messages.properties`

```properties
app.greeting=Hello, {0}!
```

`src/main/resources/i18n/problem-messages.properties`

```properties
problem.duplicate-category.title=Category already exists
problem.duplicate-category.detail=A category with name "{0}" already exists.
```

---

## Tiny tests

```java
// src/test/java/com/example/app/i18n/MessagesTest.java
package com.example.app.i18n;

import org.junit.jupiter.api.Test;
import org.springframework.context.support.StaticMessageSource;
import org.springframework.context.i18n.LocaleContextHolder;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

class MessagesTest {

  @Test
  void resolvesWithLocaleContext() {
    var sms = new StaticMessageSource();
    sms.addMessage("app.greeting", Locale.ENGLISH, "Hello, {0}!");
    LocaleContextHolder.setLocale(Locale.ENGLISH);

    var m = new Messages(sms);
    assertThat(m.msg("app.greeting", "World")).isEqualTo("Hello, World!");
  }

  @Test
  void tryMsgReturnsEmptyWhenMissing() {
    var sms = new StaticMessageSource();
    LocaleContextHolder.setLocale(Locale.ENGLISH);

    var m = new Messages(sms);
    assertThat(m.tryMsg("missing.key")).isEmpty();
  }
}
```

---

## Placement recap

```
src/main/java/com/example/app/
├─ config/
│  ├─ MessageSourceConfig.java
│  └─ MessagesConfig.java           # bean factory for Messages
└─ i18n/
   └─ Messages.java                 # the utility itself
```

This keeps the **i18n API** (`Messages`) separate from **wiring** (`MessageSourceConfig`), matching the clean split you’ve used for time.

---

## Checklist

* `Messages` compiled in `com.example.app.i18n`
* `MessagesConfig` exposes it as a bean
* Bundles present (`i18n/messages`, `i18n/problem-messages`, etc.)
* `WebConfig` locale resolver already in place (you have it)

