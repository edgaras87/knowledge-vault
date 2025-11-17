---
title: ToSelfLink
date: 2025-11-09
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing ToSelfLink in a Java Spring application, providing a MapStruct qualifier for generating canonical self-links in web-side mapping.
aliases:
    - Spring Web Support Mapping Layer - ToSelfLink Cheatsheet
---

# ToSelfLink — Keep @Mapping Expressions Tiny

`@ToSelfLink` is a **MapStruct qualifier** you can apply to small, static helper methods that produce canonical links using your `WebMappingContext` (which carries `ApiLinks`/`ProblemLinks`, locale, etc.).

You get:

- Clean `@Mapping` lines (`@ToSelfLink` instead of large `expression = "java(...)"`)
- Reuse across mappers
- Centralized link shapes

---

## Where it lives

- **Qualifier annotation:** `com.example.app.web.support.mapping.ToSelfLink`
- **Qualifiers impl (per feature or shared):** e.g.  
  `com.example.app.web.user.UserLinkQualifiers` *(feature-scoped)*  
  or `com.example.app.web.support.mapping.LinkQualifiers` *(cross-feature)*

---

## 1) Qualifier annotation

```java
// src/main/java/com/example/app/web/support/mapping/ToSelfLink.java
package com.example.app.web.support.mapping;

import org.mapstruct.Qualifier;

import java.lang.annotation.*;

@Qualifier
@Target({ ElementType.METHOD, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.CLASS)
@Documented
public @interface ToSelfLink {}
```

> Keep the annotation **framework-agnostic** (no Spring deps); MapStruct detects it via `@Qualifier`.

---

## 2) Feature-scoped qualifier helpers (User)

```java
// src/main/java/com/example/app/web/user/UserLinkQualifiers.java
package com.example.app.web.user;

import com.example.app.web.support.mapping.ToSelfLink;
import com.example.app.web.support.mapping.WebMappingContext;

import java.net.URI;

public final class UserLinkQualifiers {
  private UserLinkQualifiers() {}

  /** /api/v{n}/users/{id} */
  @ToSelfLink
  public static URI toSelf(long id, WebMappingContext ctx) {
    return ctx.apiLinks().self("users", id);
  }

  /** /api/v{n}/users */
  public static URI toCollection(WebMappingContext ctx) {
    return ctx.apiLinks().collection("users");
  }

  /** /api/v{n}/users/{id}/orders/{orderId} */
  public static URI toOrder(long userId, long orderId, WebMappingContext ctx) {
    return ctx.apiLinks().selfNested("users/" + userId + "/orders", orderId);
  }
}
```

> Add only what your mappers repeat. If helpers become generic (e.g., common paging), move them to a shared `LinkQualifiers` under `web/support/mapping/`.

---

## 3) Mapper usage

```java
// src/main/java/com/example/app/web/user/UserWebMapper.java
package com.example.app.web.user;

import com.example.app.domain.user.User;
import com.example.app.web.support.mapping.WebMappingContext;
import com.example.app.web.user.dto.response.UserResponse;
import org.mapstruct.*;

@Mapper(componentModel = "spring", uses = UserLinkQualifiers.class)
public interface UserWebMapper {

  @Mapping(target = "self", qualifiedBy = ToSelfLink.class)  // ← tiny & readable
  UserResponse toResponse(User src, @Context WebMappingContext ctx);

  // MapStruct matches ToSelfLink.toSelf(long, WebMappingContext)
  default long mapId(User src) { return src.id(); }
}
```

**DTO**

```java
// src/main/java/com/example/app/web/user/dto/response/UserResponse.java
package com.example.app.web.user.dto.response;

import java.net.URI;

public record UserResponse(long id, String email, String fullName, URI self) {}
```

**Controller (building the context)**

```java
@GetMapping("/{id}")
public UserResponse get(@PathVariable long id, java.util.Locale locale) {
  var u = service.get(id);
  var ctx = new WebMappingContext(apiLinks, problemLinks, locale, java.time.Clock.systemUTC());
  return mapper.toResponse(u, ctx);
}
```

---

## 4) Optional: shared qualifiers for common patterns

```java
// src/main/java/com/example/app/web/support/mapping/LinkQualifiers.java
package com.example.app.web.support.mapping;

import java.net.URI;

public final class LinkQualifiers {
  private LinkQualifiers() {}

  /** Generic self helper: /api/v{n}/{segment}/{id} */
  @ToSelfLink
  public static URI toSelf(String segment, long id, WebMappingContext ctx) {
    return ctx.apiLinks().self(segment, id);
  }
}
```

Use in a mapper:

```java
@Mapper(componentModel = "spring", uses = LinkQualifiers.class)
public interface CategoryWebMapper {
  @Mapping(target = "self", expression = "java(com.example.app.web.support.mapping.LinkQualifiers.toSelf(\"categories\", src.id(), ctx))")
  CategoryResponse toResponse(Category src, @Context WebMappingContext ctx);
}
```

> You can also create **more qualifiers** (`@ToCollectionLink`, `@ToOrderLink`) if that reads better.

---

## Tests (concise)

```java
// src/test/java/com/example/app/web/support/mapping/ToSelfLinkTest.java
package com.example.app.web.support.mapping;

import com.example.app.config.AppProps;
import com.example.app.web.support.errors.ProblemLinks;
import com.example.app.web.support.links.ApiLinks;
import com.example.app.web.support.links.UriFactory;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.time.Clock;
import java.util.Locale;

import static com.example.app.web.user.UserLinkQualifiers.toSelf;
import static org.assertj.core.api.Assertions.assertThat;

class ToSelfLinkTest {

  @Test
  void buildsUserSelfLink() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var ctx = new WebMappingContext(
        new ApiLinks(new UriFactory(props), props),
        new ProblemLinks(new UriFactory(props)),
        Locale.ENGLISH,
        Clock.systemUTC()
    );

    assertThat(toSelf(42L, ctx).toString())
        .isEqualTo("https://example.com/api/v1/users/42");
  }
}
```

---

## Gotchas & guardrails

* **Static, pure helpers** only—no service/repository calls.
* Keep qualifiers **feature-scoped** if they mention feature segments (`"users"`). Promote to shared only when truly generic.
* Don’t overdo qualifiers; if a one-off mapping gets unreadable, add a small default method in the mapper instead.
* Ensure your mapper includes the helper class via `@Mapper(uses = …)` or MapStruct won’t find the qualified method.

---

## Minimal checklist

* `@ToSelfLink` qualifier annotation present
* Feature qualifier helpers (e.g., `UserLinkQualifiers`) implemented as `static` methods
* Mappers reference helpers via `@Mapper(uses = …)` and `qualifiedBy = ToSelfLink.class`
* Tests assert exact URI output (includes version from `ApiLinks`)

---

## Next helper

Round out the **errors** cluster: if not yet done, wire `ProblemFactory`, `ExceptionMappings`, and finish `GlobalExceptionHandler`. Then circle back to the **http** cluster (`CacheControlPolicy`, `EtagFactory`, `ContentDispositionSafe`) for polished responses.
