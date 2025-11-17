---
title: ServletUriComponentsBuilder
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

## What it is (and why it’s useful)

`ServletUriComponentsBuilder` (Spring MVC) builds **absolute URLs** from the **current HTTP request** (scheme, host, port, context path). It saves you from manual string concatenation, handles encoding, and stays correct across environments (localhost, production, reverse proxies).

It is essentially a servlet-aware extension of `UriComponentsBuilder`, aware of your running request and servlet context.

---

## Quick glossary

* **Context path** – Base path your app is mounted at (e.g., `/myapp`).
* **Request URI** – Path of the current request (e.g., `/api/users/42`).
* **Builder** – Fluent object you add parts to (path, query, fragment) to produce a URL.
* **UriComponents** – Immutable representation of a URI (scheme/host/port/path/query).
* **Forwarded headers** – Proxy headers (`X-Forwarded-*`) telling the app the external URL.
* **Encode** – Percent-encode unsafe characters (once—and only once).

---

## Core package & location

* **Class:** `org.springframework.web.servlet.support.ServletUriComponentsBuilder`
* **Module:** `spring-webmvc`
* **Also see:** `org.springframework.web.util.UriComponentsBuilder`

---

## Typical use: building the `Location` header (201 Created)

```java
@PostMapping("/contacts")
public ResponseEntity<ContactDto> create(@RequestBody CreateContact cmd) {
  Contact saved = service.create(cmd);

  URI location = ServletUriComponentsBuilder
      .fromCurrentRequest()      // e.g., http://localhost:8080/contacts
      .path("/{id}")             // → http://.../contacts/{id}
      .buildAndExpand(saved.getId())
      .toUri();                  // encodes safely

  return ResponseEntity.created(location).body(ContactDto.from(saved));
}
```

This produces a **fully qualified URI**, correctly encoded and respecting forwarded headers when configured.

---

## Common factory methods

| Factory method                           | Base                                             | When to use                                    |
| ---------------------------------------- | ------------------------------------------------ | ---------------------------------------------- |
| `fromCurrentRequest()`                   | Full current request URI (with query)            | For related links within the same endpoint     |
| `fromCurrentRequestUri()`                | Current URI, but without query params            | When adding new query params                   |
| `fromCurrentContextPath()`               | Context root only (`scheme://host:port/context`) | For top-level links like `/health` or `/login` |
| `fromRequest(request)`                   | Explicit `HttpServletRequest`                    | When the request object is available           |
| `UriComponentsBuilder.fromPath("/path")` | No servlet dependency                            | For tests or offline link building             |

---

## Examples — creating URLs

### From current request or context

```java
String a = ServletUriComponentsBuilder
        .fromCurrentContextPath()      // https://example.com/myapp
        .path("/files/123.png")
        .toUriString();

String b = ServletUriComponentsBuilder
        .fromCurrentRequestUri()       // https://example.com/myapp/api/users/42
        .replacePath("/health")        // path becomes /health
        .replaceQuery(null)            // drop query
        .toUriString();
```

**Output:**

```
https://example.com/myapp/files/123.png
https://example.com/myapp/health
```

### From explicit `HttpServletRequest`

```java
String url = ServletUriComponentsBuilder
        .fromRequest(request)
        .replacePath("/docs")
        .replaceQuery("v=1")
        .toUriString();
// → https://example.com/myapp/docs?v=1
```

---

## Building relative or templated URIs

```java
String url = ServletUriComponentsBuilder
        .fromCurrentContextPath()
        .path("/api")
        .pathSegment("users", "{id}")  // safe segment joining + encoding
        .queryParam("verbose", "{v}")  // adds ?verbose=1
        .buildAndExpand(42, 1)         // expand placeholders
        .toUriString();

// → https://example.com/myapp/api/users/42?verbose=1
```

### Advanced transformations

```java
String url = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .replacePath("/search")
        .replaceQueryParam("q", "café") // encoded to caf%C3%A9
        .fragment("top")
        .toUriString();
```

**Output:**

```
https://example.com/myapp/search?q=caf%C3%A9#top
```

---

## Reading URI components

```java
var comps = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .build();

System.out.println(comps.getScheme());   // https
System.out.println(comps.getHost());     // example.com
System.out.println(comps.getPort());     // -1 (default)
System.out.println(comps.getPath());     // /myapp/api/users/42
System.out.println(comps.getQuery());    // active=true
```

---

## Adding or replacing query parameters

```java
URI uri = ServletUriComponentsBuilder
    .fromCurrentRequestUri()
    .queryParam("page", 2)
    .queryParam("size", 20)
    .queryParam("sort", "name,asc")
    .build()
    .toUri();
```

---

## Output conversions

```java
UriComponents comps = ServletUriComponentsBuilder.fromCurrentRequestUri().build();
URI uri = comps.toUri();              // java.net.URI
String asString = comps.toUriString(); // String
```

---

## Practical utilities

**Pagination links (RFC‑5988 style):**

```java
UriComponentsBuilder base = ServletUriComponentsBuilder.fromCurrentRequestUri();
String first = base.replaceQueryParam("page", 0).toUriString();
String next  = base.replaceQueryParam("page", page + 1).toUriString();
String prev  = base.replaceQueryParam("page", Math.max(page - 1, 0)).toUriString();

return ResponseEntity.ok()
  .header("Link",
      "<" + first + ">; rel=\"first\", " +
      "<" + next  + ">; rel=\"next\",  " +
      "<" + prev  + ">; rel=\"prev\"")
  .body(body);
```

**Self link assembler:**

```java
public static URI selfForId(Object id) {
  return ServletUriComponentsBuilder
      .fromCurrentRequestUri()
      .path("/{id}")
      .buildAndExpand(id)
      .toUri();
}
```

---

## Encoding behavior

* Path and query parts are **encoded automatically**.
* Never manually `URLEncoder.encode` before passing.
* `URI.create()` does *not* encode and will throw for illegal chars.
* Always rely on `pathSegment()` or `buildAndExpand()`.

---

## Proxy / deployment awareness

Behind reverse proxies (e.g. Nginx, Traefik):

Enable forwarded header support:

```java
@Bean
ForwardedHeaderFilter forwardedHeaderFilter() { return new ForwardedHeaderFilter(); }
```

Or in `application.properties`:

```properties
server.forward-headers-strategy=framework
```

Otherwise, you’ll see internal addresses like `http://localhost:8080` in generated links.

---

## Gotchas & anti-patterns

* **Placeholders aren’t magic:** must call `buildAndExpand(...)`.
* **Double slashes:** avoid `path("/a/").path("/b")` — use `pathSegment()`.
* **Off-request usage:** `fromCurrent*()` requires a servlet request context.
* **Null query values:** `queryParam("x", (Object)null)` produces `?x`; use `replaceQueryParam()` to remove.
* **Double encoding:** don’t call `.encode()` twice.

---

## Mini reference table

| Method                      | What it does                          | Example                       |
| --------------------------- | ------------------------------------- | ----------------------------- |
| `fromCurrentContextPath()`  | Base = scheme + host + port + context | `https://ex.com/app`          |
| `fromCurrentRequestUri()`   | Full current URI                      | `https://ex.com/app/api/u/42` |
| `path("/x")`                | Append raw path                       | `/app/x`                      |
| `pathSegment("a", "b")`     | Append encoded segments               | `/a%20b`                      |
| `queryParam("k", v)`        | Add query param                       | `?k=1`                        |
| `replacePath("/p")`         | Replace path                          | `/p`                          |
| `replaceQueryParam("k", v)` | Replace param                         | `?k=9`                        |
| `fragment("top")`           | Add fragment                          | `#top`                        |
| `buildAndExpand(...)`       | Expand templates                      | `/u/42`                       |
| `toUri()`                   | → `java.net.URI`                      |                               |
| `toUriString()`             | → String                              |                               |

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

System.out.println(profile);
System.out.println(avatar);
System.out.println(search);
```

**Output:**

```
https://api.example.com/myapp/profiles/42
https://api.example.com/myapp/files/avatars/42.png
https://api.example.com/myapp/search?q=caf%C3%A9&page=2
```

---

## Bottom line summary

* Use **`fromCurrentContextPath()`** to build top-level or absolute URLs.
* Use **`fromCurrentRequestUri()`** to tweak the current endpoint into a related one.
* Prefer **`pathSegment()`** over manual slashes.
* Enable **`ForwardedHeaderFilter`** for proxy correctness.
* Outside servlet context → use **`UriComponentsBuilder`**.

