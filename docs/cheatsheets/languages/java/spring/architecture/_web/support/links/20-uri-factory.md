---
title: UriFactory
date: 2025-11-07
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A quick reference on implementing and using a URI Factory in Spring applications to standardize URI construction and ensure consistent link generation across the application.
aliases:
    - Spring Web Support Links Layer - URI Factory Cheatsheet
---



# UriFactory — Normalize Once, Compose Everywhere

`UriFactory` does exactly two things:

1. **Normalizes** the external host from config (adds a single trailing slash, validates it’s absolute).
2. **Composes** absolute URIs by resolving paths against that host (or any provided base).

> API **versioning is not here**. It will be layered in `ApiLinks` (e.g., `/api/v1/…`).  
> This keeps `UriFactory` generic and reusable for other bases like `/problems/`, `/docs/`, etc.

---

## Where it lives

- **Class:** `com.example.app.web.support.links.UriFactory`
- **Consumes:** `com.example.app.config.AppProps` (`app.host`)
- **Used by:** `ApiLinks`, `ProblemLinks`, `PaginationLinks`, feature `*ResourceLinks`

---

## Configuration

```yaml
# src/main/resources/application.yml
app:
  host: https://example.com     # canonical external base (absolute; scheme + host)
  api:
    version: v1                 # NOTE: used later by ApiLinks, not by UriFactory
```

```java
// src/main/java/com/example/app/config/AppProps.java
package com.example.app.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

import java.net.URI;

@ConfigurationProperties(prefix = "app")
public record AppProps(
    URI host,
    Api api // present for ApiLinks; UriFactory only cares about 'host'
) {
  public record Api(String version) { }
}
```

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/links/UriFactory.java
package com.example.app.web.support.links;

import com.example.app.config.AppProps;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.util.Objects;

@Component
public class UriFactory {

  private final URI host; // normalized, always ends with '/'

  public UriFactory(AppProps props) {
    this.host = normalizeHost(props.host());
  }

  /** External host, normalized to have exactly one trailing slash. */
  public URI host() { return host; }

  /** Build a base directly under host, ending with '/'. Example: baseUnderHost("problems") -> https://example.com/problems/ */
  public URI baseUnderHost(String segment) {
    Objects.requireNonNull(segment, "segment");
    if (segment.isBlank()) return host;
    return host.resolve(ensureEndsWithSlash(trimSlashes(segment)));
  }

  /** Resolve a child path under a given base. Example: resolve(base, "users/42") -> https://…/users/42 */
  public URI resolve(URI base, String path) {
    Objects.requireNonNull(base, "base");
    Objects.requireNonNull(path, "path");
    return base.resolve(trimLeadingSlash(path));
  }

  // ---------- helpers (package-private for tests) ----------

  static URI normalizeHost(URI raw) {
    if (raw == null) throw new IllegalArgumentException("app.host must be set");
    if (!raw.isAbsolute() || raw.getScheme() == null || raw.getHost() == null) {
      throw new IllegalArgumentException("app.host must be absolute, e.g., https://example.com/");
    }
    return URI.create(ensureEndsWithSlash(raw.toString()));
  }

  static String ensureEndsWithSlash(String s) {
    return s.endsWith("/") ? s : s + "/";
  }

  static String trimSlashes(String s) {
    return s.replaceAll("^/+", "").replaceAll("/+$", "");
  }

  static String trimLeadingSlash(String s) {
    return s.replaceFirst("^/+", "");
  }
}
```

---

## Usage patterns

```java
// Build common bases
URI problemsBase = uriFactory.baseUnderHost("problems");   // https://example.com/problems/
URI docsBase     = uriFactory.baseUnderHost("docs");       // https://example.com/docs/

// Compose children
URI problemType  = uriFactory.resolve(problemsBase, "resource-not-found");
URI docPage      = uriFactory.resolve(docsBase, "errors/validation");
```

`ApiLinks` will call `uriFactory.baseUnderHost("api")` and then add `/v1/` itself.

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/links/UriFactoryTest.java
package com.example.app.web.support.links;

import com.example.app.config.AppProps;
import org.junit.jupiter.api.Test;

import java.net.URI;

import static org.assertj.core.api.Assertions.*;

class UriFactoryTest {

  @Test
  void normalizesHostAndResolvesPaths() {
    var f = new UriFactory(new AppProps(URI.create("https://example.com"), new AppProps.Api("v1")));

    assertThat(f.host().toString()).isEqualTo("https://example.com/");

    var problems = f.baseUnderHost("problems");
    assertThat(problems.toString()).isEqualTo("https://example.com/problems/");

    var type = f.resolve(problems, "duplicate-category");
    assertThat(type.toString()).isEqualTo("https://example.com/problems/duplicate-category");
  }

  @Test
  void rejectsRelativeHosts() {
    assertThatThrownBy(() ->
        new UriFactory(new AppProps(URI.create("/relative"), new AppProps.Api("v1"))))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void blankSegmentReturnsHost() {
    var f = new UriFactory(new AppProps(URI.create("https://example.com/"), new AppProps.Api("v1")));
    assertThat(f.baseUnderHost("").toString()).isEqualTo("https://example.com/");
  }
}
```

---

## Gotchas & guardrails

* **Absolute host only.** Must include scheme + host; fail fast otherwise.
* **Normalize once.** Trailing-slash handling belongs here, not in callers.
* **No servlet context.** Never use `ServletUriComponentsBuilder` in factories/helpers.
* **Compose, don’t concat.** Always `resolve(base, path)` for child URIs.
* **Keep version logic out.** Versioning lives in `ApiLinks` to keep this class generic.

---

## Minimal checklist

- `app.host` is absolute (e.g., `https://example.com`) and normalizes to `…/`.
- `baseUnderHost("segment")` returns an absolute base ending with `/`.
- `resolve(base, path)` builds correct absolute URIs.
- Unit tests assert exact strings for key cases.

---

## Next class

Proceed to **`ApiLinks`**. It will:

* take `uriFactory.baseUnderHost("api")`,
* insert the configured `app.api.version` (e.g., `v1`),
* and expose `base()`, `collection(segment)`, `self(segment, id)`, and `withQuery(...)`.
