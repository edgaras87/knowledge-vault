---
title: ApiLinks 
date: 2025-11-08
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing ApiLinks in a Java Spring application, providing a versioned API base and URI composition utilities for consistent link generation across the application.
aliases:
    - Spring Web Support Links Layer - ApiLinks Cheatsheet
---



# ApiLinks — Shared `/api/v{n}/` Base

`ApiLinks` gives the whole app a **single, normalized, versioned** API base and tiny helpers to compose resource URIs.  
It sits on top of `UriFactory` (host + base normalization). **Versioning lives here**, not in `UriFactory`.

---

## Where it lives

- **Class:** `com.example.app.web.support.links.ApiLinks`
- **Depends on:** `UriFactory` and `AppProps.api.version`
- **Used by:** feature `*ResourceLinks`, `PaginationLinks`, occasional controllers

---

## Configuration (already present)

```yaml
app:
  host: https://example.com   # absolute, normalized by UriFactory
  api:
    version: v1               # major only; becomes /api/v1/
```

`UriFactory` handles `app.host`. `ApiLinks` reads `app.api.version` and inserts it into the `/api/` path.

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/links/ApiLinks.java
package com.example.app.web.support.links;

import com.example.app.config.AppProps;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.Objects;
import java.util.function.UnaryOperator;
import java.util.regex.Pattern;

@Component
public class ApiLinks {

  private static final Pattern VERSION_SEGMENT = Pattern.compile("v\\d+"); // v1, v2, v12...

  private final UriFactory uri;
  private final String version;  // e.g., v1
  private final URI apiBase;     // e.g., https://example.com/api/v1/

  public ApiLinks(UriFactory uri, AppProps props) {
    this.uri = uri;
    this.version = validateVersion(props.api() != null ? props.api().version() : null);
    // host/api/ + version/
    this.apiBase = uri.baseUnderHost("api").resolve(this.version + "/");
  }

  /** The normalized, versioned API base (always ends with '/'). */
  public URI base() { return apiBase; }

  /** API version string (e.g., "v1"). */
  public String version() { return version; }

  /** Resolve a relative path under /api/v{n}/. Example: "users/42" → https://…/api/v1/users/42 */
  public URI resolve(String path) {
    Objects.requireNonNull(path, "path");
    return uri.resolve(apiBase, path);
  }

  /** Collection root: "users" → https://…/api/v1/users */
  public URI collection(String segment) {
    Objects.requireNonNull(segment, "segment");
    return resolve(segment);
  }

  /** Self link: ("users", 42) → https://…/api/v1/users/42 */
  public URI self(String segment, long id) {
    Objects.requireNonNull(segment, "segment");
    return resolve(segment + "/" + id);
  }

  /** Nested self link: ("users/42/orders", 9) → https://…/api/v1/users/42/orders/9 */
  public URI selfNested(String nestedSegment, long id) {
    Objects.requireNonNull(nestedSegment, "nestedSegment");
    return resolve(nestedSegment + "/" + id);
  }

  /** Build a URI with encoded query parameters using a customizer. */
  public URI withQuery(String path, UnaryOperator<UriComponentsBuilder> customizer) {
    Objects.requireNonNull(path, "path");
    Objects.requireNonNull(customizer, "customizer");
    var b = UriComponentsBuilder.fromUri(resolve(path));
    var built = customizer.apply(b).encode(StandardCharsets.UTF_8).build(true);
    return built.toUri();
  }

  // -------- helpers --------

  static String validateVersion(String v) {
    if (v == null || v.isBlank())
      throw new IllegalArgumentException("app.api.version must be set (e.g., v1)");
    if (!VERSION_SEGMENT.matcher(v).matches())
      throw new IllegalArgumentException("app.api.version must match pattern v<digits> (e.g., v1)");
    return v;
  }
}
```

---

## Usage patterns

### In feature `*ResourceLinks`

```java
// src/main/java/com/example/app/web/user/support/UserResourceLinks.java
@Component
public class UserResourceLinks {
  private final ApiLinks api;
  public UserResourceLinks(ApiLinks api) { this.api = api; }

  public URI collection() { return api.collection("users"); }
  public URI self(long id) { return api.self("users", id); }

  public URI orders(long userId) { return api.resolve("users/" + userId + "/orders"); }
  public URI order(long userId, long orderId) { return api.selfNested("users/" + userId + "/orders", orderId); }

  public URI page(int page, int size) {
    return api.withQuery("users", b -> b.queryParam("page", page).queryParam("size", size));
  }
}
```

### In controllers (light touch)

```java
@PostMapping
public ResponseEntity<Void> create(@RequestBody CreateUserRequest req) {
  var user = service.create(req);
  return ResponseEntity.created(userLinks.self(user.id())).build();
}
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/links/ApiLinksTest.java
package com.example.app.web.support.links;

import com.example.app.config.AppProps;
import org.junit.jupiter.api.Test;

import java.net.URI;

import static org.assertj.core.api.Assertions.*;

class ApiLinksTest {

  @Test
  void composesUnderVersionedApiBase() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var f = new UriFactory(props);
    var api = new ApiLinks(f, props);

    assertThat(api.base().toString()).isEqualTo("https://example.com/api/v1/");
    assertThat(api.collection("users").toString()).isEqualTo("https://example.com/api/v1/users");
    assertThat(api.self("users", 42).toString()).isEqualTo("https://example.com/api/v1/users/42");
    assertThat(api.selfNested("users/42/orders", 9).toString())
        .isEqualTo("https://example.com/api/v1/users/42/orders/9");
  }

  @Test
  void buildsWithQueryParams() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v7"));
    var f = new UriFactory(props);
    var api = new ApiLinks(f, props);

    var uri = api.withQuery("search", b -> b
        .queryParam("q", "a b")
        .queryParam("page", 1)
        .queryParam("size", 50));

    assertThat(uri.toString())
        .isEqualTo("https://example.com/api/v7/search?q=a%20b&page=1&size=50");
  }

  @Test
  void rejectsInvalidVersion() {
    var bad = new AppProps(URI.create("https://example.com"), new AppProps.Api("1"));
    var f = new UriFactory(new AppProps(URI.create("https://example.com"), new AppProps.Api("v1")));
    assertThatThrownBy(() -> new ApiLinks(f, bad))
        .isInstanceOf(IllegalArgumentException.class);
  }
}
```

---

## Gotchas & guardrails

* **No servlet context here.** Deterministic in unit tests; use `UriComponentsBuilder` only for query encoding.
* **Keep it generic.** Segments like `"users"` belong to feature `*ResourceLinks`.
* **Version once.** Changing `app.api.version` flips the entire API base; don’t sprinkle `/v1/` elsewhere.
* **Encode queries centrally.** Prefer `withQuery(...)` to avoid manual encoding mistakes.

---

## Minimal checklist

* `base()` → `https://example.com/api/v{n}/` and ends with `/`.
* `collection(segment)`, `self(segment,id)`, `selfNested(...)` cover common links.
* `withQuery(path, builder)` handles encoded query parameters.
* Feature `*ResourceLinks` delegate to `ApiLinks`; controllers don’t do string math.

---

## Next helper

With `ApiLinks` set, add feature `*ResourceLinks` and then **`PaginationLinks`** + **`LinkHeader`** to standardize navigation links and headers across list endpoints.



