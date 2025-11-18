---
title: Link Mapping
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - patterns
    - cheatsheet 
  
summary: A comprehensive guide to implementing Link Mapping Patterns using MapStruct in Java Spring applications.
aliases:
    - MapStruct Link Mapping Patterns Cheatsheet

---

# Link Mapping Patterns

REST-style APIs often include **links** in responses:

- `self` — link to this resource  
- `collection` — link to the collection it belongs to  
- `related` — links to related resources  
- `type` / `problems` — for error details (Problem Details)

These links should **not** be built in the domain layer or scattered ad-hoc across controllers.  
This file describes patterns for mapping links cleanly using helpers, context, and MapStruct.

---

## Contents

1. [Why links are a mapping concern](#1-why-links-are-a-mapping-concern)  
2. [Link helper design (ApiLinks and ProblemLinks)](#2-link-helper-design-apilinks-and-problemlinks)  
3. [Pattern 1 – Direct link mapping with context](#3-pattern-1--direct-link-mapping-with-context)  
4. [Pattern 2 – Qualifier-based link mapping](#4-pattern-2--qualifier-based-link-mapping)  
5. [Pattern 3 – Controller-driven links](#5-pattern-3--controller-driven-links)  
6. [Choosing patterns](#6-choosing-patterns)  
7. [Gotchas and guardrails](#7-gotchas-and-guardrails)  

---

## 1. Why links are a mapping concern

Links belong to the **web edge**, not the domain.  
They depend on:

- host / base URI  
- routing conventions  
- API versioning  
- current locale or representation

Because of this, they naturally live in:

- web support helpers: `ApiLinks`, `ProblemLinks`  
- web-side mapping context: `WebMappingContext`  
- web mappers: `UserWebMapper`, `CategoryWebMapper`, etc.

Domain entities should never construct HTTP URLs.

---

## 2. Link helper design (ApiLinks and ProblemLinks)

Use small, focused helpers that encode routing rules in one place.

```java
// web/support/links/ApiLinks.java
public class ApiLinks {

  private final URI base; // e.g., https://api.example.com/

  public ApiLinks(URI base) {
    this.base = base.toString().endsWith("/") ? base : URI.create(base + "/");
  }

  public URI self(String resource, long id) {
    return base.resolve(resource + "/" + id);
  }

  public URI collection(String resource) {
    return base.resolve(resource);
  }
}
```

```java
// web/support/errors/ProblemLinks.java
public class ProblemLinks {

  private final URI base; // e.g., https://api.example.com/problems/

  public ProblemLinks(URI base) {
    this.base = base.toString().endsWith("/") ? base : URI.create(base + "/");
  }

  public URI type(String slug) {
    return base.resolve(slug);
  }
}
```

These helpers are then carried by `WebMappingContext`:

```java
public record WebMappingContext(
    ApiLinks apiLinks,
    ProblemLinks problemLinks,
    Locale locale,
    Clock clock
) {}
```

---

## 3. Pattern 1 – Direct link mapping with context {#3-pattern-1--direct-link-mapping-with-context}

### Idea

Pass `WebMappingContext` (or just `ApiLinks`) as a `@Context` parameter into MapStruct mappers, and compute links in:

* expressions, or
* small helper methods

### DTO with links

```java
// web/user/dto/response/UserResponse.java
public record UserResponse(
    long id,
    String email,
    String fullName,
    URI self,
    URI orders
) {}
```

### Mapper using context

```java
// web/user/UserWebMapper.java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {

  @Mapping(target = "self",
           expression = "java(ctx.apiLinks().self(\"users\", src.id()))")
  @Mapping(target = "orders",
           expression = "java(ctx.apiLinks().self(\"users\", src.id()).resolve(\"orders\"))")
  UserResponse toResponse(User src, @Context WebMappingContext ctx);
}
```

### With helper method (cleaner)

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {

  @Mapping(target = "self",
           expression = "java(selfLink(src, ctx))")
  @Mapping(target = "orders",
           expression = "java(ordersLink(src, ctx))")
  UserResponse toResponse(User src, @Context WebMappingContext ctx);

  default URI selfLink(User src, @Context WebMappingContext ctx) {
    return ctx.apiLinks().self("users", src.id());
  }

  default URI ordersLink(User src, @Context WebMappingContext ctx) {
    return ctx.apiLinks().self("users", src.id()).resolve("orders");
  }
}
```

**When to use**

* You already have `WebMappingContext`
* Link logic is simple and per-feature
* You want link rules visible near the mapper

---

## 4. Pattern 2 – Qualifier-based link mapping {#4-pattern-2--qualifier-based-link-mapping}

### Idea

Use **qualifiers** to centralize link construction and keep `@Mapping` annotations short and reusable.

### Step 1: Qualifier annotation

```java
// web/support/mapping/ToSelfLink.java
@Qualifier
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface ToSelfLink {}
```

### Step 2: Qualifier methods

```java
// web/user/UserLinkQualifiers.java
public class UserLinkQualifiers {

  @ToSelfLink
  public static URI toSelf(User src, WebMappingContext ctx) {
    return ctx.apiLinks().self("users", src.id());
  }
}
```

### Step 3: Use in mapper

```java
@Mapper(
    config = GlobalMapperConfig.class,
    uses = UserLinkQualifiers.class
)
public interface UserWebMapper {

  @Mapping(target = "self",
           expression = "java(UserLinkQualifiers.toSelf(src, ctx))")
  UserResponse toResponse(User src, @Context WebMappingContext ctx);
}
```

You can also mix qualifiers with `qualifiedByName` or add more annotations for different link types.

**When to use**

* Many DTOs need the **same** link pattern
* You want link rules in one place per feature
* You want very readable mapping annotations

---

## 5. Pattern 3 – Controller-driven links {#5-pattern-3--controller-driven-links}

### Idea

Skip MapStruct for links and set them directly in controllers or handlers.

```java
// Controller
@GetMapping("/{id}")
public UserResponse get(@PathVariable long id, Locale locale) {
  var user = service.get(id);
  var self = apiLinks.self("users", id);
  var orders = self.resolve("orders");

  return new UserResponse(
      user.id(),
      user.email(),
      user.fullName(),
      self,
      orders
  );
}
```

**Pros**

* Extremely explicit
* No MapStruct magic
* Good for small/simple APIs

**Cons**

* Link logic spreads across controllers
* Harder to refactor base URLs or versioning rules
* Messier when many DTOs need similar links

This pattern is acceptable for tiny practice apps or one-off endpoints, but try to move to Pattern 1 or 2 as things grow.

---

## 6. Choosing patterns

### For most production-ish code

* Use **ApiLinks/ProblemLinks** helpers
* Carry them in **WebMappingContext**
* Use **Pattern 1** or **Pattern 2**

**Recommended baseline**

* `WebMappingContext` with `ApiLinks`, `ProblemLinks`, `Locale`, `Clock`
* Web mappers receive `@Context WebMappingContext`
* Link construction lives in helpers or qualifiers
* Domain has zero URL knowledge

### When Pattern 3 is enough

* Small experimental projects
* Very few endpoints
* No shared link rules

You can always refactor controllers that construct links into context-based mappers later.

---

## 7. Gotchas and guardrails

### 1) Do not build links in the domain

Domain classes should not know about:

* `http://`
* hosts / ports
* `/api/v1`
* path templates

All of that belongs to web support and mapping.

---

### 2) Normalize base URIs once

Construct `ApiLinks` and `ProblemLinks` with normalized bases:

* ensure trailing `/`
* ensure correct scheme and host
* apply API versioning here

Do not repeat `"/api/v1"` strings across mappers.

---

### 3) Keep helpers tiny

Helpers should:

* compute URIs
* not fetch data
* not perform business logic

They are pure routing rules.

---

### 4) Be careful with relative `resolve`

`URI.resolve("orders")` works only if base URIs end with `/`.
Normalize once in `ApiLinks` / `ProblemLinks` constructors.

---

### 5) Test link patterns

Write small tests asserting:

* `ApiLinks.self("users", 42)` → `https://api.example.com/users/42`
* `ProblemLinks.type("duplicate-category")` → `https://api.example.com/problems/duplicate-category`

This catches silent regressions when you change base config.

---

# Summary

Link mapping patterns separate **routing logic** from **domain logic**, and keep controllers from turning into URL factories.

Use:

* Helpers: `ApiLinks`, `ProblemLinks`
* Context: `WebMappingContext`
* MapStruct patterns:

  * **Pattern 1** — direct mapping with context + helper methods
  * **Pattern 2** — qualifier-based mapping for reusable link rules
  * **Pattern 3** — controller-driven links for very small cases

Combined with your other patterns (WebMappingContext, immutable record extra fields), this gives you a clean, repeatable way to put correct links into every response.

