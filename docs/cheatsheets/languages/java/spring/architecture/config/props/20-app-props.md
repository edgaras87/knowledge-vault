---
title: AppProps
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Bind all core app settings into a single validated record with rich types, ensuring callers receive properly parsed configuration values.
aliases:
    - Spring Config Props Layer - AppProps Cheatsheet
---

# AppProps — full, typed configuration

Bind **all** core app settings into a single validated record. Types are real (`URI`, `Path`, `InetAddress`, `DataSize`, `Duration`, `Locale`, enums, lists, maps), so callers don’t parse strings.

---

## YAML we’re binding

```yaml
app:
  host: "https://api.example.com"
  cache-dir: "/var/tmp/cache"
  ip: "192.168.1.10"

  api:
    base-path: "/api"   # changeable; normalized to leading-slash, no trailing slash
    version: v1

  pagination:
    default-size: 20
    max-size: 200

  features:
    signup: true
    metrics: true

  limits:
    uploads: 25MB
    items-per-page: 100
    timeout: 5s

  locale:
    default-tag: "en_US"

  default-roles: [USER, ADMIN]
  role-quotas:
    USER: 100
    ADMIN: 1000
  labels:
    env: "prod"
    region: "eu-central"
```

---

## Where it lives

* **Class:** `com.example.app.config.props.AppProps`
* **Used by:** `UriFactory`, `ApiLinks`, pagination policies, upload guards, i18n defaults

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/props/AppProps.java
package com.example.app.config.props;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.util.unit.DataSize;
import org.springframework.validation.annotation.Validated;

import java.net.InetAddress;
import java.net.URI;
import java.nio.file.Path;
import java.time.Duration;
import java.util.*;

@Validated
@ConfigurationProperties(prefix = "app")
public record AppProps(
    @NotNull URI host,
    @NotNull Path cacheDir,
    @NotNull InetAddress ip,

    @Valid @NotNull Api api,
    @Valid @NotNull Pagination pagination,
    @Valid @NotNull Features features,
    @Valid @NotNull Limits limits,
    @Valid @NotNull LocaleCfg locale,

    @NotNull List<Role> defaultRoles,
    @NotNull Map<Role, @Positive Integer> roleQuotas,
    @NotNull Map<@NotBlank String, @NotBlank String> labels
) {
  public AppProps {
    if (!host.isAbsolute() || host.getHost() == null) {
      throw new IllegalArgumentException("app.host must be absolute (scheme + host)");
    }
    // clamp & immutability
    defaultRoles = List.copyOf(defaultRoles == null ? List.of() : defaultRoles);
    roleQuotas   = Map.copyOf(roleQuotas   == null ? Map.of() : roleQuotas);
    labels       = Map.copyOf(labels       == null ? Map.of() : labels);

    if (pagination.defaultSize() > pagination.maxSize()) {
      throw new IllegalArgumentException("app.pagination.default-size must be <= max-size");
    }
  }

  public enum Role { USER, ADMIN }

  public record Api(
      /**
       * Leading slash, no trailing slash ("/api", "/edge", or "/" for root).
       */
      @NotBlank String basePath,
      @NotBlank @Pattern(regexp = "v[0-9]+", message = "version must be like v1, v2, …")
      String version
  ) {
    public Api {
      basePath = normalize(basePath);
    }
    private static String normalize(String s) {
      String v = (s == null ? "" : s.trim());
      if (v.isEmpty()) v = "/api";
      if (!v.startsWith("/")) v = "/" + v;
      if (v.length() > 1 && v.endsWith("/")) v = v.substring(0, v.length() - 1);
      return v;
    }
  }

  public record Pagination(@Min(1) @Max(500) int defaultSize,
                           @Min(1) @Max(1000) int maxSize) {}

  public record Features(boolean signup, boolean metrics) {}

  public record Limits(@NotNull DataSize uploads,
                       @Min(1) @Max(10000) int itemsPerPage,
                       @NotNull Duration timeout) {}

  public record LocaleCfg(@NotBlank String defaultTag) {
    public Locale defaultLocale() {
      return Locale.forLanguageTag(defaultTag.replace('_', '-'));
    }
  }
}
```

---

## Enable binding

```java
// src/main/java/com/example/app/Application.java
package com.example.app;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;

@SpringBootApplication
@ConfigurationPropertiesScan(basePackages = "com.example.app.config.props")
public class Application { }
```

---

## How other pieces consume it

* **UriFactory** → `props.host()` to produce a normalized absolute host (ending with `/`).
* **ApiLinks** → `props.api().basePath()` + `props.api().version()` → `https://…/api/v1/`.
* **PaginationLinks / policies** → `props.pagination().defaultSize()` / `maxSize()`.
* **Uploads / clients** → `props.limits().uploads()` and `timeout()`.
* **Locale defaults** → `props.locale().defaultLocale()` for i18n or formatting.

*Namespaces used elsewhere (for reference):*

* `com.example.app.web.support.uri.UriFactory`
* `com.example.app.web.support.links.ApiLinks`
* `com.example.app.web.support.links.ProblemLinks`

---

## ENV equivalents

```bash
APP_HOST=https://api.example.com
APP_CACHE_DIR=/var/tmp/cache
APP_IP=192.168.1.10
APP_API_BASE_PATH=/api
APP_API_VERSION=v1
APP_PAGINATION_DEFAULT_SIZE=20
APP_PAGINATION_MAX_SIZE=200
APP_FEATURES_SIGNUP=true
APP_FEATURES_METRICS=true
APP_LIMITS_UPLOADS=25MB
APP_LIMITS_ITEMS_PER_PAGE=100
APP_LIMITS_TIMEOUT=5s
APP_LOCALE_DEFAULT_TAG=en_US
APP_DEFAULT_ROLES[0]=USER
APP_DEFAULT_ROLES[1]=ADMIN
APP_ROLE_QUOTAS[USER]=100
APP_ROLE_QUOTAS[ADMIN]=1000
APP_LABELS_ENV=prod
APP_LABELS_REGION=eu-central
```

---

## Quick usage snippet

```java
// src/main/java/com/example/app/SomeComponent.java
package com.example.app;

import com.example.app.config.props.AppProps;
import org.springframework.stereotype.Component;

@Component
class UsesProps {
  UsesProps(AppProps props) {
    var base  = props.host();                     // https://api.example.com
    var api   = props.api();                      // basePath=/api, version=v1
    var page  = props.pagination().defaultSize(); // 20
    var limit = props.limits().uploads();         // DataSize 25MB
    var loc   = props.locale().defaultLocale();   // Locale("en","US")
  }
}
```

---

## Tests (binding + invariants)

```java
// src/test/java/com/example/app/config/props/AppPropsTest.java
package com.example.app.config.props;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThat;

class AppPropsTest {

  private ApplicationContextRunner runner(String... props) {
    return new ApplicationContextRunner()
        .withUserConfiguration(Bootstrap.class)
        .withPropertyValues(props);
  }

  @Test
  void bindsAllRichTypesAndNormalizesBasePath() {
    runner(
      "app.host=https://api.example.com",
      "app.cache-dir=/var/tmp/cache",
      "app.ip=192.168.1.10",
      "app.api.base-path=api/",       // messy on purpose
      "app.api.version=v1",
      "app.pagination.default-size=20",
      "app.pagination.max-size=200",
      "app.features.signup=true",
      "app.features.metrics=true",
      "app.limits.uploads=25MB",
      "app.limits.items-per-page=100",
      "app.limits.timeout=5s",
      "app.locale.default-tag=en_US",
      "app.default-roles[0]=USER",
      "app.default-roles[1]=ADMIN",
      "app.role-quotas[USER]=100",
      "app.role-quotas[ADMIN]=1000",
      "app.labels.env=prod",
      "app.labels.region=eu-central"
    ).run(c -> {
      var p = c.getBean(AppProps.class);
      assertThat(p.api().basePath()).isEqualTo("/api");
      assertThat(p.api().version()).isEqualTo("v1");
      assertThat(p.pagination().defaultSize()).isEqualTo(20);
      assertThat(p.limits().uploads().toMegabytes()).isEqualTo(25);
      assertThat(p.locale().defaultLocale().toLanguageTag()).isEqualTo("en-US");
      assertThat(p.defaultRoles()).containsExactly(AppProps.Role.USER, AppProps.Role.ADMIN);
      assertThat(p.roleQuotas().get(AppProps.Role.ADMIN)).isEqualTo(1000);
      assertThat(p.labels().get("region")).isEqualTo("eu-central");
    });
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(AppProps.class)
  static class Bootstrap {}
}
```

---

## Guardrails

* **Absolute `host`** only (scheme + host). Don’t infer from requests.
* **`base-path` normalization** happens once here; downstream just concatenates.
* **Keep `version` short** (`v1`, `v2`). Path versioning is coarse-grained.
* **Pagination invariant**: `default-size ≤ max-size`.
* **Wrap collections** with `List.copyOf`/`Map.copyOf` to prevent mutation.
* **Locale tags**: accept `en_US`/`en-US`, but emit IETF (`en-US`) for consistency.

---

## Minimal checklist

* `@ConfigurationProperties(prefix="app")` with validation
* Rich types wired: `URI`, `Path`, `InetAddress`, `DataSize`, `Duration`
* API base-path + version present and normalized
* Collections and enums bound correctly
* Tests prove binding + invariants
