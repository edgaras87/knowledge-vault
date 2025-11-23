---
title: FeatureResourceLinks
date: 2025-11-08
tags: 
    - spring
    - web
    - architecture
    - feature
    - support
    - links
summary: Cheatsheet for implementing feature-scoped ResourceLinks helpers in Spring Web applications.
aliases:
    - Spring Web Feature Support Layer - FeatureResourceLinks Cheatsheet
---

# *ResourceLinks — Feature-scoped example: `UserResourceLinks`

**Goal:** keep controllers free of string math.  
**Scope:** only the *User* feature. Cross-feature rules stay in `web/support/links`.

---

## Where it lives

- **Class:** `com.example.app.web.user.support.UserResourceLinks`
- **Depends on:** `com.example.app.web.support.links.ApiLinks`
- **Used by:** `UserController`, `UserWebMapper`, tests

---

## Conventions & routes

Base (versioned):  
`/api/v{n}/users`

Common endpoints this helper should cover:

- Collection: `GET /users` (optional: `?page=…&size=…`, `?q=…`)
- Create: `POST /users` → `Location: /users/{id}`
- Self: `GET /users/{id}`
- Update: `PUT /users/{id}`, `PATCH /users/{id}`
- Delete: `DELETE /users/{id}`
- Sub-resources (examples):  
  - Orders: `/users/{id}/orders`, `/users/{id}/orders/{orderId}`  
  - Addresses: `/users/{id}/addresses`, `/users/{id}/addresses/{addrId}`

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/user/support/UserResourceLinks.java
package com.example.app.web.user.support;

import com.example.app.web.support.links.ApiLinks;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.Objects;
import java.util.function.UnaryOperator;

@Component
public class UserResourceLinks {

  private final ApiLinks api;

  public UserResourceLinks(ApiLinks api) {
    this.api = api;
  }

  /** /api/v{n}/users */
  public URI collection() {
    return api.collection("users");
  }

  /** /api/v{n}/users/{id} */
  public URI self(long userId) {
    return api.self("users", userId);
  }

  /** Location for 201 Created (alias for self) */
  public URI location(long userId) {
    return self(userId);
  }

  /** /api/v{n}/users?page=&size= */
  public URI page(int page, int size) {
    return withQuery("users", b -> b.queryParam("page", page).queryParam("size", size));
  }

  /** /api/v{n}/users?q=&page=&size= (simple search pattern) */
  public URI search(String q, Integer page, Integer size) {
    return withQuery("users", b -> {
      if (q != null && !q.isBlank()) b.queryParam("q", q);
      if (page != null) b.queryParam("page", page);
      if (size != null) b.queryParam("size", size);
      return b;
    });
  }

  /** /api/v{n}/users/{id}/orders */
  public URI orders(long userId) {
    return api.resolve("users/" + userId + "/orders");
  }

  /** /api/v{n}/users/{id}/orders/{orderId} */
  public URI order(long userId, long orderId) {
    return api.selfNested("users/" + userId + "/orders", orderId);
  }

  /** /api/v{n}/users/{id}/addresses */
  public URI addresses(long userId) {
    return api.resolve("users/" + userId + "/addresses");
  }

  /** /api/v{n}/users/{id}/addresses/{addrId} */
  public URI address(long userId, long addrId) {
    return api.selfNested("users/" + userId + "/addresses", addrId);
  }

  // ---- helpers ----

  private URI withQuery(String path, UnaryOperator<UriComponentsBuilder> customizer) {
    Objects.requireNonNull(path, "path");
    Objects.requireNonNull(customizer, "customizer");
    var b = UriComponentsBuilder.fromUri(api.resolve(path));
    return customizer.apply(b).encode(StandardCharsets.UTF_8).build(true).toUri();
  }
}
```

---

## Controller usage

```java
// src/main/java/com/example/app/web/user/UserController.java (excerpt)
@RestController
@RequestMapping("/users") // NOTE: handler mapping doesn't affect link building; ApiLinks handles versioning
public class UserController {

  private final UserService service;
  private final UserResourceLinks links;

  public UserController(UserService service, UserResourceLinks links) {
    this.service = service;
    this.links = links;
  }

  @PostMapping
  public ResponseEntity<Void> create(@RequestBody CreateUserRequest req) {
    var created = service.create(req);
    return ResponseEntity.created(links.location(created.id())).build();
  }

  @GetMapping("/{id}")
  public UserResponse get(@PathVariable long id) {
    var u = service.get(id);
    return new UserResponse(u.id(), u.email(), u.fullName(), links.self(u.id()));
  }

  @GetMapping
  public ResponseEntity<List<UserListItem>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    var data = service.findPage(page, size);
    return ResponseEntity.ok()
        .header("Link", linkHeaderForUsers(page, size, data.totalPages()))
        .body(data.items());
  }

  private String linkHeaderForUsers(int page, int size, int lastPage) {
    // simple example; consider a dedicated PaginationLinks + LinkHeader helper for consistency
    var self  = links.page(page, size);
    var first = links.page(0, size);
    var prev  = links.page(Math.max(0, page - 1), size);
    var next  = links.page(Math.min(lastPage, page + 1), size);
    var last  = links.page(lastPage, size);
    return "<" + self + ">; rel=\"self\", "
        + "<" + first + ">; rel=\"first\", "
        + "<" + prev + ">; rel=\"prev\", "
        + "<" + next + ">; rel=\"next\", "
        + "<" + last + ">; rel=\"last\"";
  }
}
```

---

## DTO example (self link embedded)

```java
// src/main/java/com/example/app/web/user/dto/response/UserResponse.java
package com.example.app.web.user.dto.response;

import java.net.URI;

public record UserResponse(long id, String email, String fullName, URI self) { }
```

---

## MapStruct usage (optional)

```java
// src/main/java/com/example/app/web/support/mapping/WebMappingContext.java
package com.example.app.web.support.mapping;

import com.example.app.web.user.support.UserResourceLinks;
public record WebMappingContext(UserResourceLinks userLinks) { }
```

```java
// src/main/java/com/example/app/web/user/UserWebMapper.java
@Mapper(componentModel = "spring")
public interface UserWebMapper {

  @Mapping(target = "self", expression = "java(ctx.userLinks().self(src.id()))")
  UserResponse toResponse(User src, @Context WebMappingContext ctx);
}
```

Controller:

```java
@GetMapping("/{id}")
public UserResponse get(@PathVariable long id) {
  var u = service.get(id);
  return mapper.toResponse(u, new WebMappingContext(userLinks));
}
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/user/support/UserResourceLinksTest.java
package com.example.app.web.user.support;

import com.example.app.config.AppProps;
import com.example.app.web.support.links.ApiLinks;
import com.example.app.web.support.links.UriFactory;
import org.junit.jupiter.api.Test;

import java.net.URI;

import static org.assertj.core.api.Assertions.assertThat;

class UserResourceLinksTest {

  @Test
  void buildsStableUserUris() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var uriFactory = new UriFactory(props);
    var api = new ApiLinks(uriFactory, props);
    var links = new UserResourceLinks(api);

    assertThat(links.collection().toString())
        .isEqualTo("https://example.com/api/v1/users");
    assertThat(links.self(42).toString())
        .isEqualTo("https://example.com/api/v1/users/42");
    assertThat(links.orders(42).toString())
        .isEqualTo("https://example.com/api/v1/users/42/orders");
    assertThat(links.order(42, 9).toString())
        .isEqualTo("https://example.com/api/v1/users/42/orders/9");
  }

  @Test
  void buildsSearchAndPageUris() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v7"));
    var uriFactory = new UriFactory(props);
    var api = new ApiLinks(uriFactory, props);
    var links = new UserResourceLinks(api);

    assertThat(links.page(2, 50).toString())
        .isEqualTo("https://example.com/api/v7/users?page=2&size=50");

    assertThat(links.search("A B", 1, 25).toString())
        .isEqualTo("https://example.com/api/v7/users?q=A%20B&page=1&size=25");
  }
}
```

---

## Gotchas & guardrails

* **Keep it feature-scoped.** If logic starts to look generic (pagination headers, query encoding), promote it to `web/support/links` (e.g., `PaginationLinks`, `LinkHeader`).
* **No business logic.** Never call services or repositories here.
* **No servlet builders.** Deterministic unit tests only; `ApiLinks` + `UriComponentsBuilder` for encoding.
* **Stable paths.** Changing segment names (e.g., `"users"`) is a breaking API change—treat carefully.

---

## Minimal checklist

* `collection()`, `self(id)`, `location(id)` implemented
* Common nested routes covered (e.g., orders, addresses)
* `page(page,size)` and `search(q,page,size)` helpers provided
* Tests assert exact URIs, including version from `ApiLinks`

---

## Next helpers

Proceed to the shared navigation utilities:

* `PaginationLinks` — build `first/prev/next/last` URIs generically
* `LinkHeader` — format the RFC 5988 `Link` header from those URIs
