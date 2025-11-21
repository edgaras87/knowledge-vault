---
title: (Servlet)UriComponentsBuilder
date: 2025-10-22
tags: 
    - java
    - spring
    - spring-mvc
    - uri
    - url
    - web
    - cheatsheet
summary: Quick reference for building URIs with Spring's ServletUriComponentsBuilder
aliases:
  - ServletUriComponentsBuilder Cheatsheet
---

# 🧩 Java & Spring Web MVC — `ServletUriComponentsBuilder` Cheatsheet

> Unified quick reference: from basic Java concepts of URI building to Spring’s `ServletUriComponentsBuilder` and related utilities for safe, framework-integrated URL construction.

---
## Content

- [What it is (and why it’s useful)](#what-it-is-and-why-its-useful)
- [UriComponentsBuilder vs ServletUriComponentsBuilder](#uricomponentsbuilder-vs-servleturicomponentsbuilder)
- [Quick glossary](#quick-glossary)
- [Core package & location](#core-package--location)
- [Typical use: building the Location header (201 Created)](#typical-use-building-the-location-header-201-created)
- [Common factory methods](#common-factory-methods)
- [Examples — creating URLs](#examples--creating-urls)
- [Building relative / templated URIs](#building-relative--templated-uris)
- [Advanced transformations](#advanced-transformations)
- [Reading URI components](#reading-uri-components)
- [Adding or modifying query params](#adding-or-modifying-query-params)
- [Conversions](#conversions)
- [Proxy / deployment awareness](#proxy--deployment-awareness)
- [Gotchas & anti-patterns](#gotchas--anti-patterns)
- [Mini reference table](#mini-reference-table)
- [End-to-end example](#end-to-end-example)

## What it is (and why it’s useful)

`ServletUriComponentsBuilder` builds absolute URLs based on the **current servlet request**.  
It captures scheme, host, port, and context path automatically, ensuring the generated URL matches what the client sees — even behind proxies (when forwarded headers are enabled).

It is essentially a servlet-aware extension of `UriComponentsBuilder`.

---

## 🧭 `UriComponentsBuilder` vs `ServletUriComponentsBuilder`

Understanding these two builders is key to placing URI logic in the right layer.

### `UriComponentsBuilder` — the pure builder

A servlet-free, context-free URI builder.  
It does not know anything about the current HTTP request. You supply everything (scheme, host, path, query).

Use it when:

* you’re outside the web layer (services, utilities, scheduled tasks)
* you want predictable behavior in tests
* you want reusable URI logic without servlet coupling

Example:

```java
UriComponentsBuilder b = UriComponentsBuilder
    .fromHttpUrl("https://api.example.com")
    .path("/items/{id}")
    .queryParam("lang", "en");

URI uri = b.buildAndExpand(42).toUri();
````

---

### `ServletUriComponentsBuilder` — the request-aware builder

Knows the current `HttpServletRequest`.
Automatically uses scheme, host, port, context path, and path from the active request.

Use it when:

* you’re in a controller
* you’re generating `Location` headers
* you need URLs that reflect proxy settings
* you want high-level link assembly that adapts to environment

Example:

```java
URI location = ServletUriComponentsBuilder
    .fromCurrentRequest()
    .path("/{id}")
    .buildAndExpand(saved.getId())
    .toUri();
```

---

### Injected `UriComponentsBuilder` (controller parameter)

Spring can inject a per-request `UriComponentsBuilder`:

```java
@PostMapping("/users")
public ResponseEntity<Void> create(
        @RequestBody CreateUserInput input,
        UriComponentsBuilder builder) {

    URI loc = builder.path("/{id}").buildAndExpand(42).toUri();
    return ResponseEntity.created(loc).build();
}
```

Even though the type is generic, the instance is actually **a `ServletUriComponentsBuilder` under the hood**.

Advantages:

* avoids static calls
* easier for unit tests
* loosely coupled to Spring MVC
* still request-aware

---

## When to use what (quick rule)

* **Controllers:** use **injected `UriComponentsBuilder`**.
* **Still in controllers but no injection available:** use `ServletUriComponentsBuilder.fromCurrentRequest()`.
* **Services/domain:** don’t build URLs here; but if needed, use plain `UriComponentsBuilder`.
* **Helper classes at the web edge:** either explicit `ServletUriComponentsBuilder` or injected builder.

---

## Summary table

| Scenario                                       | Best choice                                   |
| ---------------------------------------------- | --------------------------------------------- |
| Building `Location` header                     | Injected `UriComponentsBuilder` or `Servlet*` |
| Link building inside services                  | `UriComponentsBuilder`                        |
| Building URLs unrelated to the current request | `UriComponentsBuilder`                        |
| Working behind reverse proxies                 | `ServletUriComponentsBuilder`                 |
| URL building in unit tests                     | `UriComponentsBuilder`                        |

---

## Quick glossary

* **Context path** – Base path the app is mounted at (e.g., `/myapp`).
* **Request URI** – Full path of incoming request (e.g., `/api/users/42`).
* **UriComponentsBuilder** – Fluent builder for URIs.
* **UriComponents** – The immutable result of a built URI.
* **Forwarded headers** – Tell the app the client-visible scheme/host.

---

## Core package & location {#core-package--location}

* **Class:** `org.springframework.web.servlet.support.ServletUriComponentsBuilder`
* **Module:** `spring-webmvc`
* **Related:** `org.springframework.web.util.UriComponentsBuilder`

---

## Typical use: building the `Location` header (201 Created)

```java
@PostMapping("/contacts")
public ResponseEntity<ContactDto> create(@RequestBody CreateContact cmd) {
  Contact saved = service.create(cmd);

  URI location = ServletUriComponentsBuilder
      .fromCurrentRequest()
      .path("/{id}")
      .buildAndExpand(saved.getId())
      .toUri();

  return ResponseEntity.created(location).body(ContactDto.from(saved));
}
```

This produces a fully-qualified, correctly encoded URL respecting forwarded headers.

---

## Common factory methods

| Method                                | Base                                 | When to use                              |
| ------------------------------------- | ------------------------------------ | ---------------------------------------- |
| `fromCurrentRequest()`                | Full request URI (with query)        | Same-endpoint links                      |
| `fromCurrentRequestUri()`             | Request URI without query            | Modify/replace query                     |
| `fromCurrentContextPath()`            | `<scheme>://<host>:<port>/<context>` | Top-level or unrelated links             |
| `fromRequest(request)`                | Explicit `HttpServletRequest`        | When you already have the request object |
| `UriComponentsBuilder.fromPath("/x")` | No servlet dependency                | Tests, services, utilities               |

---

## Examples — creating URLs {#examples--creating-urls}

### From current request or context

```java
String a = ServletUriComponentsBuilder
        .fromCurrentContextPath()
        .path("/files/123.png")
        .toUriString();

String b = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .replacePath("/health")
        .replaceQuery(null)
        .toUriString();
```

Outputs:

```
https://example.com/myapp/files/123.png
https://example.com/myapp/health
```

---

### Building relative / templated URIs {#building-relative--templated-uris}

```java
String url = ServletUriComponentsBuilder
    .fromCurrentContextPath()
    .path("/api")
    .pathSegment("users", "{id}")
    .queryParam("verbose", "{v}")
    .buildAndExpand(42, 1)
    .toUriString();
```

---

### Advanced transformations

```java
String url = ServletUriComponentsBuilder
    .fromCurrentRequestUri()
    .replacePath("/search")
    .replaceQueryParam("q", "café")
    .fragment("top")
    .toUriString();
```

---

### Reading URI components

```java
var comps = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .build();

comps.getScheme();
comps.getHost();
comps.getPort();
comps.getPath();
comps.getQuery();
```

---

### Adding or modifying query params

```java
URI uri = ServletUriComponentsBuilder
    .fromCurrentRequestUri()
    .queryParam("page", 2)
    .queryParam("size", 20)
    .build()
    .toUri();
```

---

### Conversions

```java
UriComponents comps = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .build();

URI uri = comps.toUri();
String s = comps.toUriString();
```

---

### Proxy / deployment awareness {#proxy--deployment-awareness}

Enable forwarded header handling:

```properties
server.forward-headers-strategy=framework
```

or

```java
@Bean
ForwardedHeaderFilter forwardedHeaderFilter() {
    return new ForwardedHeaderFilter();
}
```

Without this, URLs may appear as internal ones (`http://localhost:8080`) instead of external ones.

---

## Gotchas & anti-patterns {#gotchas--anti-patterns}

* Always call `.buildAndExpand(...)` if you have `{variables}`.
* Avoid manual slash-juggling; use `pathSegment()`.
* Don’t URL-encode manually.
* `fromCurrent*()` only works inside an active servlet request.
* Use `replaceQueryParam` to remove a param instead of passing null.

---

## Mini reference table

| Method                     | Purpose                   |
| -------------------------- | ------------------------- |
| `fromCurrentContextPath()` | Build from app root       |
| `fromCurrentRequestUri()`  | Build from current URL    |
| `path("/x")`               | Append raw path           |
| `pathSegment("a","b")`     | Append encoded segments   |
| `queryParam("k",v)`        | Add query param           |
| `replaceQueryParam("k",v)` | Replace/remove param      |
| `fragment("top")`          | Add `#fragment`           |
| `buildAndExpand()`         | Expand placeholders       |
| `toUri()`                  | Convert to `java.net.URI` |

---

## End-to-end example

```java
// Incoming: https://api.example.com/myapp/api/users/42?active=true

String profile = ServletUriComponentsBuilder
        .fromCurrentContextPath()
        .path("/profiles/{id}")
        .buildAndExpand(42)
        .toUriString();

String avatar = ServletUriComponentsBuilder
        .fromCurrentContextPath()
        .path("/files/avatars/{id}.png")
        .buildAndExpand(42)
        .toUriString();

String search = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .replacePath("/search")
        .replaceQueryParam("q", "café")
        .replaceQueryParam("page", 2)
        .toUriString();
```

Outputs:

```
https://api.example.com/myapp/profiles/42
https://api.example.com/myapp/files/avatars/42.png
https://api.example.com/myapp/search?q=caf%C3%A9&page=2
```

---

## Bottom line summary

* Use **injected `UriComponentsBuilder`** in controllers — cleanest option.
* Use **`ServletUriComponentsBuilder`** when you need explicit request-aware construction.
* Use **`UriComponentsBuilder`** everywhere else.
* Enable forwarded header awareness in production.
* Prefer `pathSegment()` to hand-written slashes.

