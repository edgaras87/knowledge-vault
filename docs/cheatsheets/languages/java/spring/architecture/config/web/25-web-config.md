---
title: WebConfig
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Centralize MVC wiring for CORS, locales, validation, content negotiation, and path-matching policies in a single WebConfig class to keep controllers clean and consistent.
aliases:
    - Spring Config Web Layer - WebConfig Cheatsheet
---


# WebConfig — MVC wiring & HTTP policy

Centralize **CORS**, **locales**, **validation**, **content negotiation**, and a few safe path-matching toggles. Keep per-controller annotations to a minimum.

---

## Where it lives

* **Class/web:** `com.example.app.web.config.WebConfig`
* **Used by:** all MVC controllers automatically

---

## Recommended defaults

* **CORS:** explicit origins, methods, headers; short dev, tight prod
* **Locales:** use `Accept-Language` with a deterministic default (from `AppProps.locale`)
* **Validation:** single `Validator` wired to your `MessageSource` (i18n errors)
* **Content negotiation:** JSON by default; no path extensions; 406 for nonsense
* **Path matching:** case-sensitive, no trailing-slash magic
* **Misc:** no hidden method filter tricks; UTF-8 everywhere

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/web/WebConfig.java
package com.example.app.web.config;

import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.validation.Validator;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.web.servlet.LocaleContextResolver;
import org.springframework.web.servlet.config.annotation.ContentNegotiationConfigurer;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.PathMatchConfigurer;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleContextResolver;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.List;
import java.util.Locale;

@Configuration
public class WebConfig implements WebMvcConfigurer {

  private final AppProps appProps;
  private final MessageSource messageSource;

  public WebConfig(AppProps appProps, MessageSource messageSource) {
    this.appProps = appProps;
    this.messageSource = messageSource;
  }

  // ----- Locale / i18n ------------------------------------------------------

  @Bean
  public LocaleContextResolver localeResolver() {
    var r = new AcceptHeaderLocaleContextResolver();
    Locale def = appProps.locale().defaultLocale(); // from your AppProps
    r.setDefaultLocale(def);
    // Optionally restrict supported locales:
    // r.setSupportedLocales(List.of(Locale.ENGLISH, Locale.GERMAN));
    return r;
  }

  @Bean
  public Validator validator() {
    var v = new LocalValidatorFactoryBean();
    v.setValidationMessageSource(messageSource);
    return v;
  }

  // ----- CORS ---------------------------------------------------------------

  @Override
  public void addCorsMappings(CorsRegistry registry) {
    // Keep origins explicit. For local dev, you might externalize these.
    registry.addMapping("/**")
        .allowedOrigins(
            "http://localhost:3000",
            "http://localhost:5173"
            // add prod origin(s) via profile-specific config
        )
        .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
        .allowedHeaders("*")
        .exposedHeaders("Location", "Link")
        .allowCredentials(false)
        .maxAge(Duration.ofMinutes(10).getSeconds());
  }

  // ----- Content negotiation ------------------------------------------------

  @Override
  public void configureContentNegotiation(ContentNegotiationConfigurer c) {
    c.defaultContentType(new MediaType(MediaType.APPLICATION_JSON, StandardCharsets.UTF_8))
     .favorParameter(false)
     .favorPathExtension(false)
     .ignoreUnknownPathExtensions(true)
     .useRegisteredExtensionsOnly(true);
  }

  // ----- Path matching (predictable URLs) -----------------------------------

  @Override
  public void configurePathMatch(PathMatchConfigurer c) {
    c.setUseTrailingSlashMatch(false); // "/users" != "/users/"
    c.setUseCaseSensitiveMatch(true);  // "/Users" != "/users"
    // If you mount feature roots with a common base, you can customize here.
  }

  // (No need to touch message converters; JacksonConfig already sets JSON policy.)
}
```

> You’ve already got `MessageSourceConfig` and `JacksonConfig`. `WebConfig` stitches them into MVC without duplicating concerns.

---

## Optional: profile-based CORS (dev vs prod)

If you want zero code changes between envs, bind a tiny `CorsProps`:

```java
// src/main/java/com/example/app/config/web/CorsProps.java
package com.example.app.web.config;

import jakarta.validation.constraints.NotNull;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import java.util.List;

@Validated
@ConfigurationProperties(prefix = "web.cors")
public record CorsProps(
    @NotNull List<String> allowedOrigins
) {}
```

YAML:

```yaml
# application.yml
web:
  cors:
    allowed-origins: ["http://localhost:3000", "http://localhost:5173"]

# application-prod.yml
web:
  cors:
    allowed-origins: ["https://app.example.com"]
```

Inject `CorsProps` into `WebConfig` and use `props.allowedOrigins().toArray(String[]::new)`.

---

## Tiny smoke test (CORS + locale)

```java
// src/test/java/com/example/app/config/web/WebConfigTest.java
package com.example.app.web.config;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Import;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.options;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.header;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest
@Import({WebConfig.class, MessageSourceConfig.class}) // your existing message source config
class WebConfigTest {

  @Autowired MockMvc mvc;
  @Autowired WebConfig config;

  @Test
  void corsPreflightIsPermitted() throws Exception {
    mvc.perform(options("/api/users")
            .header("Origin", "http://localhost:3000")
            .header("Access-Control-Request-Method", "GET"))
        .andExpect(status().isOk())
        .andExpect(header().exists("Access-Control-Allow-Origin"));
  }

  @Test
  void defaultLocaleIsSetFromProps() {
    var resolver = config.localeResolver();
    assertThat(resolver.getDefaultLocale()).isNotNull();
  }
}
```

---

## Guardrails & notes

* **CORS:** never `allowedOrigins("*")` in prod. Keep it explicit; set per-env via properties.
* **No path extensions:** URLs like `/users.json` are legacy; disable them to avoid content-type confusion.
* **Locale:** prefer `Accept-Language`; avoid session cookies or URL params for locale unless you truly need them.
* **Validation:** route all Bean Validation messages through your `MessageSource` for clean i18n.
* **ProblemDetail:** keep exception translation in `web/support/errors` (not here) so `WebConfig` stays wiring-only.

---

## Quick checklist

* `WebConfig` active, picked up by component scan
* CORS locked to known origins per environment
* ISO JSON + UTF-8 default via `ContentNegotiationConfigurer`
* Case-sensitive, no trailing-slash magic
* Locale + Validator beans hooked to your `MessageSource`


---

```yaml



```

---

## WebConfig — MVC wiring & HTTP policy (+CorsProps)

Centralize **CORS (via `CorsProps`)**, **locales**, **validation**, **content negotiation**, and **path matching**. No per-controller tweaks.

### Where it lives

* Class: `com.example.app.config.WebConfig`
* Used by: all MVC controllers

### Implementation (production-ready)

```java
// src/main/java/com/example/app/config/WebConfig.java
package com.example.app.config;

import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.validation.Validator;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.web.servlet.LocaleContextResolver;
import org.springframework.web.servlet.config.annotation.ContentNegotiationConfigurer;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.PathMatchConfigurer;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleContextResolver;

import java.nio.charset.StandardCharsets;
import java.util.Locale;

@Configuration
public class WebConfig implements WebMvcConfigurer {

  private final AppProps appProps;
  private final MessageSource messageSource;
  private final CorsProps cors;

  public WebConfig(AppProps appProps, MessageSource messageSource, CorsProps cors) {
    this.appProps = appProps;
    this.messageSource = messageSource;
    this.cors = cors;
  }

  // ----- Locale / i18n ------------------------------------------------------

  @Bean
  public LocaleContextResolver localeResolver() {
    var r = new AcceptHeaderLocaleContextResolver();
    Locale def = appProps.locale().defaultLocale(); // e.g., en-US from AppProps
    r.setDefaultLocale(def);
    // Optionally restrict supported locales:
    // r.setSupportedLocales(List.of(Locale.ENGLISH, Locale.GERMAN));
    return r;
  }

  @Bean
  public Validator validator() {
    var v = new LocalValidatorFactoryBean();
    v.setValidationMessageSource(messageSource);
    return v;
  }

  // ----- CORS (driven by CorsProps) -----------------------------------------

  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
        .allowedOrigins(cors.originsArray())
        .allowedMethods(cors.methodsArray())
        .allowedHeaders(cors.headersArray())
        .exposedHeaders(cors.exposedArray())
        .allowCredentials(cors.allowCredentials())
        .maxAge(cors.maxAge().toSeconds());
  }

  // ----- Content negotiation ------------------------------------------------

  @Override
  public void configureContentNegotiation(ContentNegotiationConfigurer c) {
    c.defaultContentType(new MediaType(MediaType.APPLICATION_JSON, StandardCharsets.UTF_8))
     .favorParameter(false)
     .favorPathExtension(false)
     .ignoreUnknownPathExtensions(true)
     .useRegisteredExtensionsOnly(true);
  }

  // ----- Path matching (predictable URLs) -----------------------------------

  @Override
  public void configurePathMatch(PathMatchConfigurer c) {
    c.setUseTrailingSlashMatch(false); // "/users" != "/users/"
    c.setUseCaseSensitiveMatch(true);  // "/Users" != "/users"
  }
}
```

### Notes that pair with this

* `MessageSourceConfig` supplies the `MessageSource` bean used by the validator.
* `JacksonConfig` already sets JSON policy (ISO dates, non-nulls, etc.).
* `CorsProps` is bound via `@ConfigurationProperties(prefix="web.cors")` and picked up by `@ConfigurationPropertiesScan`.

### Quick checklist

* `CorsProps` present and bound via properties/profiles
* `WebConfig` on classpath (component scanned)
* Locale default comes from `AppProps.locale.defaultTag`
* Content negotiation: JSON UTF-8 default, no path extensions
* Paths: case-sensitive, no trailing-slash magic
