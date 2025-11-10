---
title: CorsProps
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Centralize CORS settings into a single validated record with rich types, enabling services to enforce origins, methods, headers, credentials, and cache duration without string parsing.
aliases:
    - Spring Config Props Layer - CorsProps Cheatsheet
---

# CorsProps — typed CORS policy

Centralize **origins**, **methods**, **headers**, **credentials**, and **cache duration**. Keep dev loose, prod strict—via properties, not code edits.

---

## YAML we’re binding

```yaml
# application.yml  (dev defaults)
web:
  cors:
    allowed-origins:
      - "http://localhost:3000"
      - "http://localhost:5173"
    allowed-methods: ["GET","POST","PUT","PATCH","DELETE","OPTIONS"]
    allowed-headers: ["*"]
    exposed-headers: ["Location","Link"]
    allow-credentials: false
    max-age: 10m
```

```yaml
# application-prod.yml  (tight prod)
web:
  cors:
    allowed-origins: ["https://app.example.com"]
    allowed-methods: ["GET","POST","PUT","PATCH","DELETE","OPTIONS"]
    allowed-headers: ["Authorization","Content-Type","Accept","If-Match","If-None-Match"]
    exposed-headers: ["Location","Link","ETag"]
    allow-credentials: true
    max-age: 30m
```

---

## Where it lives

* **Class:** `com.example.app.config.CorsProps`
* **Used by:** `WebConfig.addCorsMappings`

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/CorsProps.java
package com.example.app.config;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;

@Validated
@ConfigurationProperties(prefix = "web.cors")
public record CorsProps(
    @NotNull List<@NotBlank String> allowedOrigins,
    @NotNull List<@NotBlank String> allowedMethods,
    @NotNull List<@NotBlank String> allowedHeaders,
    @NotNull List<@NotBlank String> exposedHeaders,
    @NotNull Boolean allowCredentials,
    @NotNull Duration maxAge
) {
  public CorsProps {
    // Defensive copies + basic invariants
    allowedOrigins = copyOrEmpty(allowedOrigins);
    allowedMethods = copyOrEmpty(allowedMethods);
    allowedHeaders = copyOrEmpty(allowedHeaders);
    exposedHeaders = copyOrEmpty(exposedHeaders);

    if (allowedOrigins.isEmpty()) {
      throw new IllegalArgumentException("web.cors.allowed-origins must contain at least one origin");
    }
    if (maxAge.isNegative() || maxAge.isZero()) {
      throw new IllegalArgumentException("web.cors.max-age must be positive");
    }
    // Warn-on-purpose: using "*" with allowCredentials=true is invalid per CORS spec
    if (allowCredentials && allowedOrigins.stream().anyMatch("*"::equals)) {
      throw new IllegalArgumentException("cannot use wildcard origin when allow-credentials=true");
    }
  }

  private static <T> List<T> copyOrEmpty(List<T> in) {
    return in == null ? List.of() : List.copyOf(in);
  }

  // Convenience helpers for WebConfig
  public String[] originsArray() { return allowedOrigins.toArray(String[]::new); }
  public String[] methodsArray() { return allowedMethods.toArray(String[]::new); }
  public String[] headersArray() { return allowedHeaders.toArray(String[]::new); }
  public String[] exposedArray() { return exposedHeaders.toArray(String[]::new); }
}
```

---

## Wire into `WebConfig`

```java
// src/main/java/com/example/app/config/WebConfig.java
package com.example.app.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

  private final AppProps appProps; // you already have this
  private final CorsProps cors;

  public WebConfig(AppProps appProps, CorsProps cors) {
    this.appProps = appProps;
    this.cors = cors;
  }

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

  // ...rest of your WebConfig (locale/validator/content-negotiation/etc.)
}
```

> Ensure `@ConfigurationPropertiesScan` is enabled in `Application` so `CorsProps` is bound.

---

## ENV equivalents

```bash
WEB_CORS_ALLOWED_ORIGINS[0]=http://localhost:3000
WEB_CORS_ALLOWED_ORIGINS[1]=http://localhost:5173

WEB_CORS_ALLOWED_METHODS[0]=GET
WEB_CORS_ALLOWED_METHODS[1]=POST
WEB_CORS_ALLOWED_METHODS[2]=PUT
WEB_CORS_ALLOWED_METHODS[3]=PATCH
WEB_CORS_ALLOWED_METHODS[4]=DELETE
WEB_CORS_ALLOWED_METHODS[5]=OPTIONS

WEB_CORS_ALLOWED_HEADERS[0]=*
WEB_CORS_EXPOSED_HEADERS[0]=Location
WEB_CORS_EXPOSED_HEADERS[1]=Link

WEB_CORS_ALLOW_CREDENTIALS=false
WEB_CORS_MAX_AGE=10m
```

---

## Tiny smoke test

```java
// src/test/java/com/example/app/config/CorsPropsTest.java
package com.example.app.config;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class CorsPropsTest {

  private ApplicationContextRunner run(String... p) {
    return new ApplicationContextRunner()
        .withUserConfiguration(Bootstrap.class)
        .withPropertyValues(p);
  }

  @Test
  void wildcardWithCredentialsIsRejected() {
    assertThatThrownBy(() -> run(
        "web.cors.allowed-origins[0]=*",
        "web.cors.allowed-methods[0]=GET",
        "web.cors.allowed-headers[0]=*",
        "web.cors.exposed-headers[0]=Location",
        "web.cors.allow-credentials=true",
        "web.cors.max-age=10m"
    ).run(c -> c.getBean(CorsProps.class))).hasMessageContaining("wildcard origin");
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(CorsProps.class)
  static class Bootstrap {}
}
```

---

## Guardrails & tips

* **Never** ship `allowed-origins: ["*"]` in prod. Be explicit; use profiles to switch.
* If you **must** support credentials (cookies/Authorization), **no wildcard** origins—list them.
* Keep `allowed-headers` tight in prod (drop `"*"`), at least to `Authorization, Content-Type, Accept, If-None-Match, If-Match`.
* Cache preflights with a sensible `max-age` (dev short; prod longer).
* Keep all policy in properties—zero code changes between environments.

---

Match `SecurityHeadersFilter` (adds `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`, etc.) to round out the web edge.

---
