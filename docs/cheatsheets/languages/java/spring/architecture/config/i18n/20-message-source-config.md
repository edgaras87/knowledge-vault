---
title: MessageSourceConfig
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Define a centralized MessageSource configuration for your Spring application to manage internationalization (i18n) and validation messages, ensuring consistent message loading, caching, and locale handling across the application.
aliases:
    - Spring Config Layer - MessageSourceConfig Cheatsheet
---

# MessageSourceConfig — i18n & validation messages

One place to define how your app loads **human-readable messages**: validation errors, RFC-7807 problem titles, e-mail templates, etc. Keep messages externalized, cacheable, and testable.

---

## Where it lives

* **Class:** `com.example.app.config.MessageSourceConfig`
* **Used by:** Bean Validation (via `WebConfig`’s `LocalValidatorFactoryBean`), exception translators, mail templates, any code that needs `MessageSource`

---

## File layout (resources)

```text
src/main/resources/i18n/
├─ messages.properties              # default (en)
├─ messages_de.properties
├─ problem-messages.properties      # RFC-7807 titles/detail texts
├─ problem-messages_de.properties
└─ validation.properties            # optional: bean validation overrides per locale
```

> You can keep validation keys in `messages*.properties` too; separating is just organization.

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/MessageSourceConfig.java
package com.example.app.config;

import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ReloadableResourceBundleMessageSource;

@Configuration
public class MessageSourceConfig {

  @Bean
  public MessageSource messageSource() {
    var ms = new ReloadableResourceBundleMessageSource();
    ms.setBasenames(
        "classpath:i18n/messages",
        "classpath:i18n/problem-messages",
        "classpath:i18n/validation"
    );
    ms.setDefaultEncoding("UTF-8");

    // Don’t fall back to the host/JVM locale; use our explicit default
    ms.setFallbackToSystemLocale(false);

    // In dev, you can set low cache seconds for hot reload; in prod prefer longer
    ms.setCacheMillis(-1); // use properties below to control via profiles if desired

    // When a code is missing, return the code itself instead of throwing
    ms.setUseCodeAsDefaultMessage(true);

    return ms;
  }
}
```

---

## Pairing with WebConfig (already done)

Your `WebConfig` wires the `LocalValidatorFactoryBean` to this `MessageSource`, so Bean Validation messages resolve from your bundles:

```java
// in WebConfig
@Bean
public Validator validator() {
  var v = new LocalValidatorFactoryBean();
  v.setValidationMessageSource(messageSource);
  return v;
}
```

---

## Properties per profile (dev vs prod)

Use Spring’s built-in props to tweak caching without touching code:

```yaml
# application.yml (dev)
spring:
  messages:
    basename: i18n/messages,i18n/problem-messages,i18n/validation
    encoding: UTF-8
    cache-duration: 1s     # hot reload-ish for dev (classpath reload still needed)
    fallback-to-system-locale: false
    use-code-as-default-message: true

# application-prod.yml
spring:
  messages:
    basename: i18n/messages,i18n/problem-messages,i18n/validation
    encoding: UTF-8
    cache-duration: 1h      # stable in prod
    fallback-to-system-locale: false
    use-code-as-default-message: true
```

> If you define `spring.messages.*`, you can skip the Java bean entirely. Keeping the bean gives you explicit control and works fine alongside the properties—Spring merges settings.

---

## Message formats & examples

### 1) Regular app text

```properties
# i18n/messages.properties
app.greeting=Hello, {0}!
app.items.count=You have {0} item.|You have {0} items.
```

Java usage:

```java
var text = messageSource.getMessage("app.greeting",
    new Object[]{"Edgaras"}, locale);  // → "Hello, Edgaras!"
```

### 2) Validation messages (Bean Validation)

```properties
# i18n/validation.properties
javax.validation.constraints.NotBlank.message=must not be blank
size.user.name=Name must be between {min} and {max} characters
```

Annotate:

```java
public record CreateUserRequest(
  @jakarta.validation.constraints.NotBlank(message = "{javax.validation.constraints.NotBlank.message}")
  String email,

  @jakarta.validation.constraints.Size(min = 3, max = 50, message = "{size.user.name}")
  String name
) {}
```

### 3) ProblemDetail (RFC-7807) texts

```properties
# i18n/problem-messages.properties
problem.duplicate-category.title=Category already exists
problem.duplicate-category.detail=A category with name "{0}" already exists.
```

Controller advice:

```java
var title  = ms.getMessage("problem.duplicate-category.title", null, locale);
var detail = ms.getMessage("problem.duplicate-category.detail", new Object[]{name}, locale);
```

---

## Locale behavior

* **Default locale** comes from `AppProps.locale.defaultTag` via `WebConfig`’s `AcceptHeaderLocaleContextResolver`.
* `fallbackToSystemLocale=false` ensures consistent English (or your chosen default) when a translation is missing.
* To restrict languages, set `r.setSupportedLocales(...)` in `WebConfig`.

---

## Plurals & parameter rules

* Spring uses **`MessageFormat`** under the hood. For plural logic, you can:

  * Use simple two-form split with pipes (“one|many”), and choose in code, **or**
  * Use ICU MessageFormat via an extra library if you need rich plural rules.
* Escape single quotes: write `don''t` to render `don't`.

---

## Tiny smoke test

```java
// src/test/java/com/example/app/config/MessageSourceConfigTest.java
package com.example.app.config;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class MessageSourceConfigTest {

  @Autowired org.springframework.context.MessageSource ms;

  @Test
  void resolvesDefaultMessage() {
    var msg = ms.getMessage("app.greeting", new Object[]{"World"}, Locale.ENGLISH);
    assertThat(msg).contains("Hello, World");
  }

  @Test
  void returnsCodeWhenMissing() {
    var msg = ms.getMessage("missing.key", null, Locale.ENGLISH);
    assertThat(msg).isEqualTo("missing.key");
  }
}
```

---

## Checklist

* Basenames defined: `i18n/messages`, `i18n/problem-messages`, `i18n/validation`
* UTF-8 encoding
* No fallback to system locale
* Cache tuned per environment
* `WebConfig` validator wired to this `MessageSource`
* Keys present for validation + problems; missing keys return the code

