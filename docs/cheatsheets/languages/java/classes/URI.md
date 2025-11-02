---
title: URI 
date: 2025-11-01
tags: 
    - java
    - standard-library
    - networking
    - best-practices
    - classes
summary: Your definitive cheatsheet for using `java.net.URI` effectively. Learn how to build, parse, and manipulate URIs safely and correctly in Java applications.
aliases:
  - Java URI Cheatsheet
---

# 🔗 `java.net.URI` — Precise, Immutable Identifiers

> **Essence:** `URI` is an **immutable** parser/builder for Uniform Resource Identifiers.
> It models pieces of an identifier (scheme, authority, path, query, fragment), does **no I/O**, and can **resolve/relativize/normalize** paths safely.

---

## 1) The anatomy of a URI

```
scheme ":" [//authority] path ["?" query] ["#" fragment]
authority = [userinfo "@"] host [ ":" port ]
```

Example:

```
https://alice:secret@example.com:8443/api/v1/users?q=bob#top
└────┬─┘ └──────┬──────┘└─────────authority──────────┘└path┘ └query┘ └fragment┘
 scheme     userinfo             host       port
```

**Opaque vs hierarchical**

* **Hierarchical:** has `//` (e.g., `https://...`) → has path, can `resolve/relativize`.
* **Opaque:** no `/` after scheme (e.g., `mailto:joe@example.com`, `urn:isbn:...`) → no path semantics.

---

## 2) Getting/inspecting parts

```java
URI u = URI.create("https://example.com:8443/a/b%20c?x=1&y=2#frag");

u.getScheme();          // "https"
u.getUserInfo();        // null (if present: "alice:secret")
u.getHost();            // "example.com"
u.getPort();            // 8443  (−1 if absent)
u.getPath();            // "/a/b c"            (decoded view)
u.getRawPath();         // "/a/b%20c"          (encoded view)
u.getQuery();           // "x=1&y=2"
u.getRawQuery();        // "x=1&y=2"           (if encoded, you see raw)
u.getFragment();        // "frag"
u.isAbsolute();         // true (has scheme)
u.isOpaque();           // false
u.getAuthority();       // "example.com:8443"
u.getSchemeSpecificPart(); // "//example.com:8443/a/b%20c?x=1&y=2"
```

> Use `getRaw*` when you care about exact percent-encoding; `get*` returns decoded characters.

---

## 3) Building URIs correctly (no broken encoding)

### A) Using multi-arg constructor (lets the JDK encode for you)

```java
URI u = new URI(
  "https",                // scheme
  null,                   // userInfo
  "example.com",          // host
  8443,                   // port
  "/a/b c",               // path (unencoded input OK)
  "q=a%2Bb&lang=en",      // query (give pre-encoded OR build carefully)
  "top"                   // fragment
);
// https://example.com:8443/a/b%20c?q=a%2Bb&lang=en#top
```

### B) For **opaque** URIs (e.g., mailto)

```java
URI m = new URI("mailto", "joe@example.com", null);
// "mailto:joe@example.com"
```

### C) If you must start from a string

```java
URI u = URI.create("https://example.com/a/b%20c?q=1#x"); // throws IAE if invalid
```

> Avoid manual string concatenation for paths. Let the constructor encode path segments.
> For complex query building, assemble the query string with careful encoding of **values** (see §6).

---

## 4) Resolve / relativize / normalize (path math)

```java
URI base = URI.create("https://example.com/app/");
URI rel  = URI.create("../img/logo.png");
base.resolve(rel);      // https://example.com/img/logo.png

URI target = URI.create("https://example.com/a/b/c");
base.relativize(target); // "a/b/c" (relative from base)

URI messy = URI.create("https://x/y/./z/../a");
messy.normalize();       // https://x/y/a
```

> Works only for **hierarchical** URIs.

---

## 5) Hostnames, IPv6, and IDNs

```java
URI v6 = new URI("http", null, "[2001:db8::1]", 8080, "/api", null, null);
// http://[2001:db8::1]:8080/api

// Internationalized domain:
URI idn = URI.create("https://münich.example/straße");
idn.toASCIIString();  // punycode host + percent-encoded path
```

**Rule:** IPv6 hosts must be bracketed `[addr]`.

---

## 6) Query parameters (safe construction)

**Do not** use `URLEncoder` on a whole query string; it’s for HTML form encoding of **parameter values**.

```java
String q = "q=" + URLEncoder.encode("a+b c", StandardCharsets.UTF_8)
          + "&lang=en";
URI u = new URI("https", "example.com", "/search", q, null);
```

If you’re in Spring, prefer `UriComponentsBuilder` for ergonomic, safe query building.

---

## 7) URI vs URL vs URLConnection

| Type                           | I/O?        | Represents                | Notes                              |
| ------------------------------ | ----------- | ------------------------- | ---------------------------------- |
| `URI`                          | ❌           | Identifier only           | Parsing, composition, math         |
| `URL`                          | ⚠️ can open | Location (protocol-bound) | Legacy, mixes ID with I/O          |
| `HttpClient` / `URLConnection` | ✅           | Network access            | Build requests; pass `URI` to them |

Convert when you must:

```java
URL url = u.toURL(); // may throw MalformedURLException if not absolute
```

---

## 8) Equality, normalization, case

```java
URI a = URI.create("HTTP://EXAMPLE.com/%7Ealice");
URI b = URI.create("http://example.com/~alice");

a.equals(b);           // false (character-by-character)
a.normalize().equals(b.normalize()); // still may be false (host case-insensitive, path not normalized by %7E vs ~)

a.toASCIIString().equalsIgnoreCase(b.toASCIIString()); // better, but not perfect
```

**Takeaway:** `URI#equals` is strict. For logical equality, normalize and compare components you care about.

---

## 9) Common operations cookbook

**Join base + segment (without double slashes)**

```java
URI base = URI.create("https://api.example.com/users/");
URI next = base.resolve("42"); // https://api.example.com/users/42
```

**Append query param**

```java
String q = "page=1&size=20";
URI u = new URI("https", "api.example.com", "/items", q, null);
```

**Strip fragment**

```java
URI noFrag = new URI(u.getScheme(), u.getSchemeSpecificPart(), null);
```

**Replace path but keep origin**

```java
URI replaced = new URI(u.getScheme(), u.getUserInfo(), u.getHost(), u.getPort(),
                       "/health", null, null);
```

---

## 10) Exceptions and validation

* `new URI(...)` throws `URISyntaxException` (checked) for invalid syntax.
* `URI.create(...)` throws `IllegalArgumentException` (unchecked).
* Typical bad inputs: unescaped spaces in **single-string** constructor, malformed IPv6, illegal chars in `host`.

---

## 11) Performance & immutability

* `URI` is **immutable** and thread-safe. Cache freely.
* Parsing is cheap; the heavy part is your own encoding/decoding and network I/O (which `URI` doesn’t do).

---

## 12) Pitfalls (and fixes)

| Pitfall                                                | Why it hurts                     | Do instead                                       |
| ------------------------------------------------------ | -------------------------------- | ------------------------------------------------ |
| Hand-concatenating `"/a" + "/" + b`                    | Double slashes, missed encoding  | Use multi-arg constructor + `resolve`            |
| Using `URLEncoder` on entire URI                       | Produces invalid `%3A//` etc.    | Encode **values** only, or let `URI` encode path |
| Forgetting brackets for IPv6                           | Parser treats `:` as port        | Use `[2001:db8::1]`                              |
| Comparing URIs with `equals` expecting “same resource” | Too strict; doesn’t canonicalize | Normalize and/or compare components              |
| Assuming `getPath()` returns encoded string            | It’s decoded view                | Use `getRawPath()` for exact bytes               |

---

## 13) Minimal patterns to memorize

**String → URI (validate)**

```java
URI u = new URI("https://example.com/a b");  // throws URISyntaxException
```

**Safe build (JDK encodes path)**

```java
URI u = new URI("https", null, "example.com", -1, "/a b", "q=a%2Bb", null);
```

**Base + relative**

```java
URI base = URI.create("https://ex.com/root/");
URI full = base.resolve("child?id=1");
```

**Relativize**

```java
URI root = URI.create("https://ex.com/a/");
URI tgt  = URI.create("https://ex.com/a/b/c");
root.relativize(tgt); // "b/c"
```

---

## 14) Where it plugs into your stack

* **`HttpClient` (Java 11+)**: `HttpRequest.newBuilder(URI).build()`
* **Spring (MVC/WebClient)**: `URI` everywhere; for building use `UriComponentsBuilder`.
* **JAX-RS**: `UriBuilder` mirrors the same ideas.

---

## 15) Mental model

```
[URI]  — pure identifier, no I/O
  • immutable components (scheme, authority, path, query, fragment)
  • safe composition (constructors) and math (resolve/relativize/normalize)
  • dual views: raw (encoded) vs decoded
  • feeds higher-level HTTP clients and frameworks
```

