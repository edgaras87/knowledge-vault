---
title: Content
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: An index of cheat sheets covering various aspects of the Spring configuration layer, including property binding, framework customization, cross-cutting infrastructure, and best practices for organizing configuration code.
aliases:
    - Spring Config Layer - Content Cheatsheet
---



# Configuration Layer — Index

The `config/` directory is your project’s **control room**.

Not a dumping ground, not a graveyard of random annotations — a small, curated place where you bind settings, initialize frameworks, customize serialization, and define global policy.

When the domain is the truth, and the web layer is the boundary, the configuration layer is the **dial panel** that makes the whole system predictable.

---

## What lives here

Anything that configures **the application itself** but doesn’t belong to business logic or the web-edge.

### 1) Property binding (external configuration → typed records)

These are your `@ConfigurationProperties` classes, for example:

* `AppProps` — canonical host, API version, feature flags
* `SecurityProps` — JWT expirations, public/private key paths
* `StorageProps` — upload limits, bucket names
* `MailProps` — SMTP host, ports, toggles

Their job is simple: turn YAML/env text into safe, typed Java records.

### 2) Framework wiring and customization

Beans that tune Spring Boot’s engine:

* Jackson config (`WebConfig`, or `JacksonConfig`)
* Validation configuration (global `Validator`)
* CORS policy
* Locale & MessageSource configuration
* Filters/interceptors not tied to a feature

### 3) Cross-cutting infrastructure

Reusable pieces used across multiple layers:

* Caching configuration
* HTTP client configuration
* ObjectMapper modules
* Date/time serialization conventions
* Cross-service clients

Not domain logic. Not HTTP parsing. Just runtime wiring.

### 4) Auto-configuration glue

Tiny configuration components that create beans used everywhere:

* `UriFactory` wired with `AppProps.host`
* `MessageSource` basenames
* `Clock` bean for testable time

---

## What must *not* live here

* Controllers → `web/`
* Exception translation → `web/support/errors`
* Persistence/JPA settings inside business modules → use `infra/persistence/config`
* Business services → `app/`
* Domain rules → `domain/`

If configuration starts depending on business logic, something has drifted into the uncanny valley.

---

## Typical inhabitants

| Category                   | Examples                                 | Responsibility                              |
| -------------------------- | ---------------------------------------- | ------------------------------------------- |
| **Properties binding**     | `AppProps`, `SecurityProps`, `MailProps` | Convert YAML/env → typed Java               |
| **Serialization settings** | `JacksonConfig`, `WebConfig`             | ObjectMapper modules, naming strategies     |
| **Internationalization**   | `MessageSourceConfig`                    | i18n basenames, locale config               |
| **Clock/time**             | `ClockConfig`                            | Provide consistent `Clock` across app/tests |
| **CORS/HTTP**              | `CorsConfig`                             | Allowed origins, methods, headers           |
| **Infrastructure hooks**   | `ClientConfig`, `CacheConfig`            | Shared clients, caches, metrics             |

---

## Example structure

```text
src/main/java/com/example/app/config/
├─ AppProps.java                # typed application settings (host, api.version)
├─ WebConfig.java               # Jackson/CORS/Locale configuration
├─ MessageSourceConfig.java     # i18n basenames and locale resolver
├─ ClockConfig.java             # central Clock bean
├─ SecurityProps.java           # JWT or OAuth-related settings
└─ ClientConfig.java            # external HTTP clients (if any)
```

---

## Example: canonical property binding

```java
// src/main/java/com/example/app/config/AppProps.java
package com.example.app.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

import java.net.URI;

@ConfigurationProperties(prefix = "app")
public record AppProps(
    URI host,
    Api api
) {
  public record Api(String version) {}
}
```

```yaml
# application.yml
app:
  host: https://example.com
  api:
    version: v1
```

---

## Example: Jackson configuration

```java
// src/main/java/com/example/app/config/WebConfig.java
package com.example.app.config;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilderCustomizer;

@Configuration
public class WebConfig {

  @Bean
  public Jackson2ObjectMapperBuilderCustomizer jsonCustomizer() {
    return builder -> {
      builder.modules(new JavaTimeModule());
      builder.featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
      builder.defaultPropertyInclusion(JsonInclude.Include.NON_NULL);
    };
  }
}
```

---

## Example: MessageSource configuration

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
         "classpath:i18n/problem-messages",
         "classpath:i18n/messages"
     );
     ms.setDefaultEncoding("UTF-8");
     ms.setFallbackToSystemLocale(false);
     return ms;
  }
}
```

---

## Example: Clock

```java
// src/main/java/com/example/app/config/ClockConfig.java
package com.example.app.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Clock;

@Configuration
public class ClockConfig {

  @Bean
  public Clock clock() {
    return Clock.systemUTC();
  }
}
```

Your services and mappers now receive a stable, testable timestamp source.

---

## Why this layer exists

* Keeps configuration away from business and web-edge code
* Centralizes runtime tuning
* Makes behavior reproducible across environments
* Guarantees single source of truth for settings
* Allows controllers and services to be clean, focused, boring — in the good way

When someone asks *“where is X configured?”* — the answer should be here.

---

## Checklist

* All external settings bind through `@ConfigurationProperties`
* Serialization conventions defined once
* Clocks, locales, MessageSource live here
* No business logic
* No controllers
* No persistence logic

---

## Subpages

* `app-props.md` — all about typed config
* `webconfig-jackson.md` — serialization policy
* `message-source.md` — i18n and problem texts
* `clock.md` — time abstraction
* `cors-config.md` — network exposure rules

The configuration layer remains the quiet hero that keeps the system predictable.

