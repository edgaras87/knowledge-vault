---
title: WebMappingContext
date: 2025-11-09
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing WebMappingContext in a Java Spring application, providing a context carrier for web-side mapping with links, locale, and clock support.
aliases:
    - Spring Web Support Mapping Layer - WebMappingContext Cheatsheet
---

# WebMappingContext — Context for Web-Side Mapping

MapStruct (and manual mappers) often need **a little web-edge context** to produce perfect DTOs: links, locale-aware strings, and stable timestamps. `WebMappingContext` is a tiny carrier for those *inputs*, keeping mapping code clean and testable.

**What it is not:** it’s not a service locator and doesn’t call repositories or perform business logic.

---

## Where it lives

- **Class:** `com.example.app.web.support.mapping.WebMappingContext`
- **Used by:** web mappers (DTO ←→ web models), occasionally controllers
- **Provides:** `ApiLinks`, `ProblemLinks`, `Locale`, `Clock` (or `ZonedDateTime.now(clock)`), lightweight formatters

> Keep **global MapStruct config** in `infra/mapping/MapStructConfig.java`.  
> `WebMappingContext` is web-edge specific (links/locale), so it belongs here.

---

## Implementation (production-ready, immutable record)

```java
// src/main/java/com/example/app/web/support/mapping/WebMappingContext.java
package com.example.app.web.support.mapping;

import com.example.app.web.support.links.ApiLinks;
import com.example.app.web.support.errors.ProblemLinks;

import java.time.Clock;
import java.util.Locale;
import java.util.Objects;

/**
 * Minimal, immutable inputs for web-side mapping.
 * - No business logic, no DB access.
 * - Pure carriers to keep mappers deterministic and HTTP-aware (links, locale).
 */
public record WebMappingContext(
    ApiLinks apiLinks,
    ProblemLinks problemLinks,
    Locale locale,
    Clock clock
) {
  public WebMappingContext {
    Objects.requireNonNull(apiLinks, "apiLinks");
    Objects.requireNonNull(problemLinks, "problemLinks");
    Objects.requireNonNull(locale, "locale");
    Objects.requireNonNull(clock, "clock");
  }

  /** Convenience for default/English contexts in tests. */
  public static WebMappingContext of(ApiLinks apiLinks, ProblemLinks prob, Locale locale) {
    return new WebMappingContext(apiLinks, prob, locale, Clock.systemUTC());
  }
}
```

### Optional light helpers (if you prefer)

If you routinely need lightweight formatting, you can wrap a couple of common ones without importing heavy libs:

```java
// Add to WebMappingContext if needed (example):
public String yesNo(boolean value) {
  return value ? "yes" : "no"; // or localize via MessageSource (inject titles/keys elsewhere)
}
```

Keep helpers tiny; anything larger belongs in a dedicated formatter bean injected *into the context* at construction time (but avoid making the context a dependency graph).

---

## Typical mapper usage (MapStruct)

### DTO with a self link and a localized title

```java
// src/main/java/com/example/app/web/user/dto/response/UserResponse.java
package com.example.app.web.user.dto.response;
import java.net.URI;

public record UserResponse(long id, String email, String fullName, URI self, String statusTitle) {}
```

```java
// src/main/java/com/example/app/web/user/UserWebMapper.java
package com.example.app.web.user;

import com.example.app.domain.user.User; // your domain model
import com.example.app.web.support.mapping.WebMappingContext;
import com.example.app.web.user.dto.response.UserResponse;
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface UserWebMapper {

  @Mapping(target = "self",
           expression = "java(ctx.apiLinks().self(\"users\", src.id()))")
  @Mapping(target = "statusTitle",
           expression = "java(localizeStatus(src, ctx))")
  UserResponse toResponse(User src, @Context WebMappingContext ctx);

  // default method keeps heavy expressions out of @Mapping annotations
  default String localizeStatus(User src, @Context WebMappingContext ctx) {
    // Example: naive english mapping; replace with MessageSource-backed text if desired
    return src.active() ? "Active" : "Inactive";
  }
}
```

### Controller passes the context

```java
@RestController
@RequestMapping("/users")
public class UserController {

  private final UserService service;
  private final UserWebMapper mapper;
  private final ApiLinks api;
  private final ProblemLinks problems;

  public UserController(UserService service, UserWebMapper mapper, ApiLinks api, ProblemLinks problems) {
    this.service = service;
    this.mapper = mapper;
    this.api = api;
    this.problems = problems;
  }

  @GetMapping("/{id}")
  public UserResponse get(@PathVariable long id, java.util.Locale locale) {
    var user = service.get(id);
    var ctx = new WebMappingContext(api, problems, locale, java.time.Clock.systemUTC());
    return mapper.toResponse(user, ctx);
  }
}
```

> You can also create a small factory method if you dislike `new` in controllers:
> `WebMappingContextFactory` that builds `WebMappingContext` from injected `ApiLinks`, `ProblemLinks`, and `LocaleResolver`.

---

## Using qualifiers with the context (optional)

For repeated link patterns, define a qualifier to keep annotations terse:

```java
// src/main/java/com/example/app/web/support/mapping/ToSelfLink.java
package com.example.app.web.support.mapping;
import org.mapstruct.Qualifier;
import java.lang.annotation.*;

@Qualifier
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface ToSelfLink {}
```

```java
// src/main/java/com/example/app/web/user/UserLinkQualifiers.java
package com.example.app.web.user;

import com.example.app.web.support.mapping.ToSelfLink;
import com.example.app.web.support.mapping.WebMappingContext;

import java.net.URI;

public class UserLinkQualifiers {
  @ToSelfLink
  public static URI toSelf(long id, WebMappingContext ctx) {
    return ctx.apiLinks().self("users", id);
  }
}
```

```java
@Mapper(componentModel = "spring", uses = UserLinkQualifiers.class)
public interface UserWebMapper {
  @Mapping(target = "self", expression = "java(UserLinkQualifiers.toSelf(src.id(), ctx))")
  UserResponse toResponse(User src, @Context WebMappingContext ctx);
}
```

---

## Tests (JUnit + AssertJ) — deterministic and fast

```java
// src/test/java/com/example/app/web/support/mapping/WebMappingContextTest.java
package com.example.app.web.support.mapping;

import com.example.app.config.AppProps;
import com.example.app.web.support.errors.ProblemLinks;
import com.example.app.web.support.links.ApiLinks;
import com.example.app.web.support.links.UriFactory;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.time.Clock;
import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

class WebMappingContextTest {

  @Test
  void carriesLinksAndLocaleAndClock() {
    var props = new AppProps(URI.create("https://example.com"), new AppProps.Api("v1"));
    var uri = new UriFactory(props);
    var api = new ApiLinks(uri, props);
    var problems = new ProblemLinks(uri);

    var ctx = new WebMappingContext(api, problems, Locale.ENGLISH, Clock.systemUTC());
    assertThat(ctx.apiLinks().base().toString()).isEqualTo("https://example.com/api/v1/");
    assertThat(ctx.problemLinks().base().toString()).isEqualTo("https://example.com/problems/");
    assertThat(ctx.locale()).isEqualTo(Locale.ENGLISH);
    assertThat(ctx.clock()).isNotNull();
  }
}
```

---

## Gotchas & guardrails

* **No business logic.** The context should never call services or repositories.
* **Keep it immutable.** Use a record and pass it down; don’t mutate in mappers.
* **Small surface.** Only include what multiple mappers need (links, locale, clock). Anything else: pass as explicit method params or add a dedicated formatter bean.
* **Framework-light.** Don’t inject `HttpServletRequest` or `UriComponentsBuilder` here; links come from `ApiLinks` / `ProblemLinks`.

---

## Minimal checklist

* `WebMappingContext` record with `ApiLinks`, `ProblemLinks`, `Locale`, `Clock`
* Mappers accept `@Context WebMappingContext`
* Controllers build/passthrough context (or use a tiny factory)
* Tests assert bases/locale/clock are present and correct

---

## Next helper

Add a tiny **`ToSelfLink` qualifier** (already shown) and consider a `DateTimeFormatters` bean if you often render timestamps in responses (inject that bean into the context constructor).

