---
title: CacheControlPolicy
date: 2025-11-09
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing CacheControlPolicy in a Java Spring application, providing header-ready cache directives for effective HTTP caching strategies.
aliases:
    - Spring Web Support Http Layer - CacheControlPolicy Cheatsheet
---

# CacheControlPolicy — Header-ready cache directives

A tiny, framework-light helper that produces **`Cache-Control`** header values for common API needs:

- `no-store` (sensitive responses)
- `no-cache` (must revalidate; allow storage)
- `public/private` with `max-age`
- `immutable` for content that never changes within its TTL
- `stale-while-revalidate` / `stale-if-error` (CDN/proxy friendly)
- `s-maxage` for shared caches

It returns **header strings** so you can use it with `ResponseEntity` or any HTTP stack.  
ETags and `Last-Modified` are handled by `EtagFactory` / `DateTimeFormatters` elsewhere.

---

## Where it lives

- **Class:** `com.example.app.web.support.http.CacheControlPolicy`
- **Used by:** controllers, download endpoints, static metadata endpoints, list/detail resources

---

## Implementation (production-ready, zero Spring deps)

```java
// src/main/java/com/example/app/web/support/http/CacheControlPolicy.java
package com.example.app.web.support.http;

import java.time.Duration;
import java.util.LinkedHashSet;
import java.util.Objects;
import java.util.Set;
import java.util.function.UnaryOperator;

/**
 * Compose Cache-Control header values for common API scenarios.
 * Returns header-ready strings like: "public, max-age=3600, immutable".
 */
public final class CacheControlPolicy {

  private CacheControlPolicy() {}

  // ---- BASIC PRESETS ----

  /** For sensitive responses: never store. */
  public static String noStore() {
    return "no-store";
  }

  /** Force revalidation each time (can be stored but must be revalidated). */
  public static String noCache() {
    return "no-cache";
  }

  /** Private caches only (per-user), with max-age and must-revalidate. */
  public static String privateMaxAge(Duration ttl) {
    return compose("private", maxAge(ttl), "must-revalidate");
  }

  /** Public caches allowed (CDNs), with max-age. */
  public static String publicMaxAge(Duration ttl) {
    return compose("public", maxAge(ttl));
  }

  /** Strong hint to clients/CDNs that content won't change during TTL. */
  public static String publicImmutable(Duration ttl) {
    return compose("public", maxAge(ttl), "immutable");
  }

  /** Public caches with different TTL for shared caches (CDN) vs browsers. */
  public static String publicMaxAgeSMaxAge(Duration ttlClient, Duration ttlShared) {
    return compose("public", maxAge(ttlClient), sMaxAge(ttlShared));
  }

  // ---- COMPOSERS ----

  /** Add stale-while-revalidate to an existing policy string. */
  public static String withStaleWhileRevalidate(String base, Duration swr) {
    return append(base, staleWhileRevalidate(swr));
  }

  /** Add stale-if-error to an existing policy string. */
  public static String withStaleIfError(String base, Duration sie) {
    return append(base, staleIfError(sie));
  }

  /** Free-form composer for advanced cases. */
  public static String customize(String base, UnaryOperator<Set<String>> mutator) {
    Objects.requireNonNull(base, "base");
    var set = split(base);
    mutator.apply(set);
    return join(set);
  }

  // ---- LOW-LEVEL TOKENS ----

  public static String maxAge(Duration d) { return "max-age=" + toSeconds(d); }
  public static String sMaxAge(Duration d) { return "s-maxage=" + toSeconds(d); }
  public static String staleWhileRevalidate(Duration d) { return "stale-while-revalidate=" + toSeconds(d); }
  public static String staleIfError(Duration d) { return "stale-if-error=" + toSeconds(d); }

  // ---- internals ----

  private static String compose(String... parts) {
    var set = new LinkedHashSet<String>();
    for (var p : parts) if (p != null && !p.isBlank()) set.add(p);
    return join(set);
  }

  private static String append(String base, String part) {
    var set = split(base);
    if (part != null && !part.isBlank()) set.add(part);
    return join(set);
  }

  private static Set<String> split(String base) {
    var set = new LinkedHashSet<String>();
    if (base != null && !base.isBlank()) {
      for (var p : base.split("\\s*,\\s*")) {
        if (!p.isBlank()) set.add(p.trim());
      }
    }
    return set;
  }

  private static String join(Set<String> parts) {
    return String.join(", ", parts);
  }

  private static long toSeconds(Duration d) {
    Objects.requireNonNull(d, "duration");
    if (d.isNegative()) return 0;
    return d.getSeconds();
  }
}
```

---

## Usage patterns

### 1) Immutable JSON for 5 minutes (CDN-friendly)

```java
var cc = CacheControlPolicy.publicImmutable(Duration.ofMinutes(5));
// "public, max-age=300, immutable"

return ResponseEntity.ok()
  .header("Cache-Control", cc)
  .body(dto);
```

### 2) Paginated list, revalidate but allow brief staleness

```java
var cc = CacheControlPolicy.publicMaxAge(Duration.ofSeconds(30));
cc = CacheControlPolicy.withStaleWhileRevalidate(cc, Duration.ofSeconds(30));
// "public, max-age=30, stale-while-revalidate=30"

return ResponseEntity.ok()
  .header("Cache-Control", cc)
  .body(page.items());
```

### 3) User-specific dashboard (private cache only)

```java
var cc = CacheControlPolicy.privateMaxAge(Duration.ofSeconds(0));
// "private, max-age=0, must-revalidate"
return ResponseEntity.ok().header("Cache-Control", cc).body(dto);
```

### 4) Separate TTLs for CDN and browsers

```java
var cc = CacheControlPolicy.publicMaxAgeSMaxAge(Duration.ofMinutes(1), Duration.ofMinutes(10));
// "public, max-age=60, s-maxage=600"
```

### 5) Combine with `Last-Modified` / `ETag`

```java
var cc = CacheControlPolicy.publicMaxAge(Duration.ofMinutes(1));
return ResponseEntity.ok()
  .header("Cache-Control", cc)
  .header("Last-Modified", com.example.app.web.support.mapping.DateTimeFormatters.rfc1123(lastModifiedInstant))
  .eTag(etag) // from EtagFactory (next helper)
  .body(dto);
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/http/CacheControlPolicyTest.java
package com.example.app.web.support.http;

import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

class CacheControlPolicyTest {

  @Test
  void presetsProduceExpectedStrings() {
    assertThat(CacheControlPolicy.noStore()).isEqualTo("no-store");
    assertThat(CacheControlPolicy.noCache()).isEqualTo("no-cache");
    assertThat(CacheControlPolicy.publicImmutable(Duration.ofMinutes(5)))
        .isEqualTo("public, max-age=300, immutable");
    assertThat(CacheControlPolicy.privateMaxAge(Duration.ofSeconds(0)))
        .isEqualTo("private, max-age=0, must-revalidate");
  }

  @Test
  void addsStaleDirectives() {
    var cc = CacheControlPolicy.publicMaxAge(Duration.ofSeconds(30));
    cc = CacheControlPolicy.withStaleWhileRevalidate(cc, Duration.ofSeconds(30));
    cc = CacheControlPolicy.withStaleIfError(cc, Duration.ofMinutes(1));
    assertThat(cc).isEqualTo("public, max-age=30, stale-while-revalidate=30, stale-if-error=60");
  }

  @Test
  void sMaxAgeForSharedCaches() {
    var cc = CacheControlPolicy.publicMaxAgeSMaxAge(Duration.ofMinutes(1), Duration.ofMinutes(10));
    assertThat(cc).isEqualTo("public, max-age=60, s-maxage=600");
  }
}
```

---

## Gotchas & guardrails

* **Pick a sensible default per endpoint**; don’t over-cache dynamic JSON. Start with `public, max-age=30, stale-while-revalidate=30` for lists that update frequently.
* **`immutable`** is a strong promise: only use when resource content truly won’t change within TTL (IDs, versioned assets, snapshot responses).
* **Private data** (user-specific) should be `private` (or `no-store` for highly sensitive).
* **ETag/Last-Modified** complement TTLs and enable conditional requests (`If-None-Match`, `If-Modified-Since`).
* **Durations** below zero are clamped to 0 seconds.

---

## Minimal checklist

* Use `public/private` deliberately
* Set `max-age` (and `s-maxage` if behind a CDN)
* Consider `stale-while-revalidate` to smooth reloads
* Add `immutable` only when safe
* Pair with `ETag` and/or `Last-Modified` for freshness validation

---

## Next helper

Wire **`EtagFactory`** to compute stable entity tags, and **`ContentDispositionSafe`** to generate safe download filenames.

