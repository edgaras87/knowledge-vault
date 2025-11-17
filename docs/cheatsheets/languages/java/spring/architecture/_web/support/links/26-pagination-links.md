---
title: PaginationLinks
date: 2025-11-08
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing PaginationLinks in a Java Spring application, providing stable navigation URIs for paginated resources.
aliases:
    - Spring Web Support Links Layer - PaginationLinks Cheatsheet
---




# PaginationLinks — Stable Navigation URIs

Keep pagination math and query encoding out of controllers. `PaginationLinks` sits on top of `ApiLinks` and produces **absolute, versioned** URIs:

- `self`, `first`, `prev`, `next`, `last`
- Optional extra query params (search, filters, sorts)

Pair this with `LinkHeader` to emit an RFC 5988 `Link` header.

---

## Where it lives

- **Class:** `com.example.app.web.support.links.PaginationLinks`
- **Depends on:** `com.example.app.web.support.links.ApiLinks`
- **Used by:** controllers, feature `*ResourceLinks` (if you prefer), tests

---

## Design

- Page indexing assumed **0-based** (Spring Data style).  
- We require `totalPages` (or `lastPage`) to build `last/next`.  
- We never touch servlet/request context.  
- Queries are encoded via `UriComponentsBuilder`.

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/links/PaginationLinks.java
package com.example.app.web.support.links;

import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.*;
import java.util.function.UnaryOperator;

@Component
public class PaginationLinks {

  private final ApiLinks api;

  public PaginationLinks(ApiLinks api) {
    this.api = api;
  }

  /** Build a page URI under /api/v{n}/{segment}?page=&size= plus optional extras. */
  public URI page(String segment, int page, int size, UnaryOperator<UriComponentsBuilder> extras) {
    Objects.requireNonNull(segment, "segment");
    var b = UriComponentsBuilder.fromUri(api.collection(segment))
        .queryParam("page", page)
        .queryParam("size", size);
    if (extras != null) b = extras.apply(b);
    return b.encode(StandardCharsets.UTF_8).build(true).toUri();
  }

  /** Overload without extras. */
  public URI page(String segment, int page, int size) {
    return page(segment, page, size, null);
  }

  /** Build page URI for nested segments, e.g., users/42/orders. */
  public URI pageNested(String nestedSegment, int page, int size, UnaryOperator<UriComponentsBuilder> extras) {
    Objects.requireNonNull(nestedSegment, "nestedSegment");
    var b = UriComponentsBuilder.fromUri(api.resolve(nestedSegment))
        .queryParam("page", page)
        .queryParam("size", size);
    if (extras != null) b = extras.apply(b);
    return b.encode(StandardCharsets.UTF_8).build(true).toUri();
  }

  public URI pageNested(String nestedSegment, int page, int size) {
    return pageNested(nestedSegment, page, size, null);
  }

  /**
   * Navigation map for a flat collection:
   * keys: self, first, prev, next, last
   */
  public Map<String, URI> nav(String segment, int page, int size, int totalPages, UnaryOperator<UriComponentsBuilder> extras) {
    var p = clamp(page, totalPages);
    var last = Math.max(0, totalPages - 1);

    var map = new LinkedHashMap<String, URI>(5);
    map.put("self",  page(segment, p, size, extras));
    map.put("first", page(segment, 0, size, extras));
    map.put("prev",  page(segment, Math.max(0, p - 1), size, extras));
    map.put("next",  page(segment, Math.min(last, p + 1), size, extras));
    map.put("last",  page(segment, last, size, extras));
    return map;
  }

  /** Navigation map for nested collections, e.g., users/42/orders */
  public Map<String, URI> navNested(String nestedSegment, int page, int size, int totalPages, UnaryOperator<UriComponentsBuilder> extras) {
    var p = clamp(page, totalPages);
    var last = Math.max(0, totalPages - 1);

    var map = new LinkedHashMap<String, URI>(5);
    map.put("self",  pageNested(nestedSegment, p, size, extras));
    map.put("first", pageNested(nestedSegment, 0, size, extras));
    map.put("prev",  pageNested(nestedSegment, Math.max(0, p - 1), size, extras));
    map.put("next",  pageNested(nestedSegment, Math.min(last, p + 1), size, extras));
    map.put("last",  pageNested(nestedSegment, last, size, extras));
    return map;
  }

  // ----- helpers -----

  static int clamp(int page, int totalPages) {
    if (totalPages <= 0) return 0;
    var last = totalPages - 1;
    if (page < 0) return 0;
    if (page > last) return last;
    return page;
  }
}
```

---

## Usage in a controller

```java
@RestController
@RequestMapping("/users")
class UserController {

  private final UserService service;
  private final PaginationLinks paging;

  UserController(UserService service, PaginationLinks paging) {
    this.service = service;
    this.paging = paging;
  }

  @GetMapping
  ResponseEntity<List<UserListItem>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size,
      @RequestParam(required = false) String q) {

    var result = service.find(q, page, size); // returns items + totalPages
    var rels = paging.nav("users", page, size, result.totalPages(),
        b -> (q == null || q.isBlank()) ? b : b.queryParam("q", q));

    var linkHeader = rels.entrySet().stream()
        .map(e -> "<" + e.getValue() + ">; rel=\"" + e.getKey() + "\"")
        .collect(java.util.stream.Collectors.joining(", "));

    return ResponseEntity.ok()
        .header("Link", linkHeader)
        .body(result.items());
  }
}
```

> If you want a reusable header builder, plug in `LinkHeader` later to format the `rels` map.

---

## Example with nested resources

```java
// /api/v1/users/{id}/orders?page=&size=&status=
var rels = paging.navNested("users/" + userId + "/orders", page, size, totalPages,
    b -> (status == null ? b : b.queryParam("status", status)));
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/links/PaginationLinksTest.java
package com.example.app.web.support.links;

import com.example.app.config.AppProps;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class PaginationLinksTest {

  @Test
  void buildsFlatNavLinks() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var f = new UriFactory(props);
    var api = new ApiLinks(f, props);
    var paging = new PaginationLinks(api);

    var rels = paging.nav("users", 2, 50, 10, b -> b.queryParam("q", "A B"));
    assertThat(rels.get("self").toString())
        .isEqualTo("https://example.com/api/v1/users?page=2&size=50&q=A%20B");
    assertThat(rels.get("first").toString())
        .isEqualTo("https://example.com/api/v1/users?page=0&size=50&q=A%20B");
    assertThat(rels.get("last").toString())
        .isEqualTo("https://example.com/api/v1/users?page=9&size=50&q=A%20B");
  }

  @Test
  void buildsNestedNavLinks() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v3"));
    var f = new UriFactory(props);
    var api = new ApiLinks(f, props);
    var paging = new PaginationLinks(api);

    var rels = paging.navNested("users/42/orders", 0, 20, 1, null);
    assertThat(rels.get("self").toString())
        .isEqualTo("https://example.com/api/v3/users/42/orders?page=0&size=20");
    assertThat(rels.get("next").toString())
        .isEqualTo("https://example.com/api/v3/users/42/orders?page=0&size=20"); // only one page → next==self
  }

  @Test
  void clampsOutOfRange() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var paging = new PaginationLinks(new ApiLinks(new UriFactory(props), props));

    var rels = paging.nav("users", 999, 10, 3, null);
    assertThat(rels.get("self").toString())
        .isEqualTo("https://example.com/api/v1/users?page=2&size=10");
  }
}
```

---

## Gotchas & guardrails

* **Encode searches/filters via the `extras` builder**; don’t concatenate `?q=…`.
* **Always provide `totalPages`** (or `lastPage`) to compute `next/last`.
* **Clamp** page numbers—never generate negative or past-the-end links.
* **Keep it generic.** If you find feature-specific rules creeping in, move them to the feature’s `*ResourceLinks` and call `PaginationLinks` from there.

---

## Minimal checklist

* `page(segment, page, size[, extras])` returns absolute, versioned URIs
* `nav(...)` provides `self/first/prev/next/last`
* Nested variant exists for paths like `users/{id}/orders`
* Tests assert exact string output with version and encoded queries

---

## Next helper

Wire **`LinkHeader`** to format the `rels` map into a standards-compliant header:

```
<Link ...>; rel="self", <Link ...>; rel="next", …
```


## Existing Spring pagination options

Spring has pagination helpers, but only if you opt into specific stacks. Your `PaginationLinks` still fills a real gap.

Here’s the lay of the land:

**Spring Data (vanilla MVC)**

- Gives you `Pageable`, `Page<T>`, and request params like `?page=&size=&sort=`.
- It does **not** build navigation links for you. You either: 
    - Put paging metadata in the body yourself, or
    - Hand-roll `Link` headers. There’s no out-of-the-box RFC 5988/8288 header builder.
  
**Spring HATEOAS (add-on)**

* Provides `PagedModel<T>` and `PagedResourcesAssembler<T>`.
* It auto-adds `self/next/prev/first/last` **in the body** as HAL (or another hypermedia format).
* Links are usually derived from the **current request** (servlet context, path, query). Absolute vs relative depends on configuration; versioned `/api/v{n}/…` bases are on you.
* If you’re not returning hypermedia bodies (you return plain lists + headers), HATEOAS is overkill.

**Spring Data REST**

* Exposes repositories automatically with HAL responses and built-in pagination links in the body.
* Great for quick CRUD, but you’ve been building a layered app with hand-crafted endpoints; Data REST likely isn’t your path.

**Why your helper still makes sense**

- You want **absolute, versioned** URIs under `/api/v{n}/…`, independent of servlet context, and you prefer **RFC Link headers** rather than HAL bodies. None of the above gives you all that out of the box.
- `UriComponentsBuilder` can build links, but there’s no opinionated, reusable component that:
    - clamps page numbers,
    - knows your versioned API base,
    - and returns a neat `self/first/prev/next/last` map.

* Your `PaginationLinks` + `LinkHeader` stays tiny, testable, and matches your `ApiLinks/UriFactory` layer.

If you wanted the “existing” option anyway

* Add Spring HATEOAS and return `PagedModel<UserListItemResponse>` via `PagedResourcesAssembler`:

  ```java
  @GetMapping
  public PagedModel<UserListItemResponse> list(Pageable pageable) {
    Page<User> page = service.find(pageable);
    return assembler.toModel(
      page.map(mapper::toUserListItemResponse),
      linkTo(methodOn(UserController.class).list(pageable)).withSelfRel()
    );
  }
  ```
* You’ll get body-embedded links, not RFC `Link` headers; and they’re derived from the incoming request, not from your versioned `ApiLinks`.

**Bottom line**

* If you want HAL hypermedia: use Spring HATEOAS and skip your helper.
* If you want plain JSON bodies + **standards-compliant `Link` header** + **stable `/api/v{n}` bases** + **no servlet coupling**: your `PaginationLinks` is exactly the right abstraction.
