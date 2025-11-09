---
title: EtagFactory
date: 2025-11-09
tags:
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing EtagFactory in a Java Spring application, providing strong and weak ETag generation along with utilities for handling conditional requests.
aliases:
    - Spring Web Support Http Layer - EtagFactory Cheatsheet
---

# EtagFactory — Strong/Weak tags, zero servlet magic

ETags make GETs cheaper and idempotent writes safer. This helper gives you **deterministic** ETag values and tiny utilities to evaluate `If-None-Match`.

- **Strong** ETags for byte-identical payloads
- **Weak** ETags for semantically equivalent representations
- Hashing via SHA-256 → **base64url** (opaque, safe), then quoted per RFC
- Helpers from **version fields**, **Instants**, **bytes/strings/UUID**
- `matchesIfNoneMatch(...)` to decide **304 Not Modified**

It returns **header-ready strings**, so you can drop them into `ResponseEntity#eTag(String)`.

---

## Where it lives

- **Class:** `com.example.app.web.support.http.EtagFactory`
- **Used by:** controllers, download endpoints, list/detail resources
- **Pairs with:** `CacheControlPolicy` and `DateTimeFormatters` (`Last-Modified`)

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/http/EtagFactory.java
package com.example.app.web.support.http;

import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.time.Instant;
import java.util.*;

/**
 * Build strong/weak ETags and evaluate If-None-Match headers.
 * Produces header-ready values: "\"tag\"" or "W/\"tag\"".
 */
public final class EtagFactory {
  private EtagFactory() {}

  // --------- STRONG TAGS (byte-identical) ---------

  /** Strong ETag from bytes (SHA-256 -> base64url, quoted). */
  public static String strong(byte[] bytes) {
    Objects.requireNonNull(bytes, "bytes");
    return quote(b64(sha256(bytes)));
  }

  /** Strong ETag from string (UTF-8), for small JSON/text if you already have it. */
  public static String strong(String payloadUtf8) {
    Objects.requireNonNull(payloadUtf8, "payloadUtf8");
    return strong(payloadUtf8.getBytes(StandardCharsets.UTF_8));
  }

  /** Strong ETag from UUID (e.g., immutable content ids). */
  public static String strong(UUID id) {
    Objects.requireNonNull(id, "id");
    var bb = ByteBuffer.allocate(16)
        .putLong(id.getMostSignificantBits())
        .putLong(id.getLeastSignificantBits())
        .array();
    return quote(b64(sha256(bb)));
  }

  // --------- WEAK TAGS (semantic equality) ---------

  /** Weak ETag from a monotonically increasing version (e.g., row version). */
  public static String weak(long version) {
    return prefixWeak(quote(Long.toUnsignedString(version)));
  }

  /** Weak ETag from Instant (e.g., updatedAt). Second precision is usually enough. */
  public static String weak(Instant updatedAt) {
    Objects.requireNonNull(updatedAt, "updatedAt");
    // Use epochSecond + nano to avoid collisions if your clock has higher precision
    var val = updatedAt.getEpochSecond() + ":" + updatedAt.getNano();
    return prefixWeak(quote(val));
  }

  /** Weak ETag from (version + size) tuple (cheap change detector). */
  public static String weak(long version, long size) {
    var val = Long.toUnsignedString(version) + ":" + Long.toUnsignedString(size);
    return prefixWeak(quote(val));
  }

  /** Convert any strong tag (already quoted) to weak (adds W/ prefix if missing). */
  public static String toWeak(String etag) {
    Objects.requireNonNull(etag, "etag");
    return isWeak(etag) ? etag : prefixWeak(etag);
  }

  // --------- CONDITIONALS ---------

  /**
   * Evaluate If-None-Match. Returns true if the request condition matches (i.e., you should return 304).
   * Handles: "*", comma-separated tags, weak-vs-strong comparison semantics.
   */
  public static boolean matchesIfNoneMatch(String ifNoneMatchHeader, String currentEtag) {
    if (ifNoneMatchHeader == null || ifNoneMatchHeader.isBlank()) return false;
    Objects.requireNonNull(currentEtag, "currentEtag");

    // "*" matches anything
    if (ifNoneMatchHeader.trim().equals("*")) return true;

    // Parse header list
    var candidates = parseEtags(ifNoneMatchHeader);
    for (var cand : candidates) {
      if (etagEquals(cand, currentEtag)) return true;
    }
    return false;
  }

  // --------- UTILS ---------

  /** True if ETag has W/ prefix (case-sensitive per RFC). */
  public static boolean isWeak(String etag) {
    return etag != null && etag.startsWith("W/");
  }

  /** Quote raw tag token if not already quoted or weak-quoted. */
  public static String quote(String token) {
    Objects.requireNonNull(token, "token");
    if (token.startsWith("\"") || token.startsWith("W/\"")) return token;
    return '"' + token + '"';
  }

  // ----- internals -----

  private static String prefixWeak(String quoted) {
    return quoted.startsWith("W/") ? quoted : "W/" + quoted;
  }

  private static byte[] sha256(byte[] bytes) {
    try {
      var md = MessageDigest.getInstance("SHA-256");
      return md.digest(bytes);
    } catch (Exception e) {
      throw new IllegalStateException("SHA-256 not available", e);
    }
  }

  private static String b64(byte[] bytes) {
    // web-safe, opaque; remove padding for shorter tokens
    return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
  }

  private static boolean etagEquals(String a, String b) {
    // RFC: weak comparison ignores weakness; strong requires exact match including weakness/quotes
    // For If-None-Match we use weak comparison: W/"x" equals "x" (and vice versa).
    var coreA = core(a);
    var coreB = core(b);
    return Objects.equals(coreA, coreB);
  }

  private static String core(String etag) {
    if (etag == null) return null;
    var s = etag.trim();
    if (s.startsWith("W/")) s = s.substring(2);
    if (s.startsWith("\"") && s.endsWith("\"") && s.length() >= 2) {
      s = s.substring(1, s.length() - 1);
    }
    return s;
  }

  private static List<String> parseEtags(String header) {
    var list = new ArrayList<String>();
    int i = 0, n = header.length();
    while (i < n) {
      // skip spaces and commas
      while (i < n && (header.charAt(i) == ' ' || header.charAt(i) == ',')) i++;
      if (i >= n) break;

      // optional W/
      int start = i;
      boolean weak = false;
      if (i + 1 < n && header.charAt(i) == 'W' && header.charAt(i + 1) == '/') {
        weak = true; i += 2;
      }

      // quoted-string
      if (i < n && header.charAt(i) == '"') {
        int j = i + 1;
        for (; j < n; j++) {
          if (header.charAt(j) == '"' && header.charAt(j - 1) != '\\') break;
        }
        if (j >= n) break; // malformed; stop
        var q = header.substring(i, j + 1);
        list.add(weak ? "W/" + q : q);
        i = j + 1;
      } else {
        // token until comma/space
        int j = i;
        while (j < n && header.charAt(j) != ',' && header.charAt(j) != ' ') j++;
        var token = header.substring(i, j).trim();
        if (!token.isEmpty()) list.add(weak ? "W/\"" + token + "\"" : "\"" + token + "\"");
        i = j + 1;
      }
    }
    return list;
  }
}
```

---

## Usage patterns

### 1) Strong tag for exact-bytes JSON

```java
var json = objectMapper.writeValueAsBytes(dto); // your canonical encoding
var etag = EtagFactory.strong(json);

return ResponseEntity.ok()
  .eTag(etag)
  .header("Cache-Control", CacheControlPolicy.publicMaxAge(Duration.ofSeconds(30)))
  .body(dto);
```

### 2) Weak tag for row version / updatedAt

```java
var etag = EtagFactory.weak(entity.getVersion());      // or EtagFactory.weak(entity.getUpdatedAt())
return ResponseEntity.ok().eTag(etag).body(dto);
```

### 3) Conditional GET → 304 Not Modified

```java
@GetMapping("/{id}")
public ResponseEntity<?> get(@PathVariable long id,
                             @RequestHeader(value = "If-None-Match", required = false) String inm) {
  var u = service.get(id);
  var etag = EtagFactory.weak(u.getUpdatedAt()); // or strong(bytes)
  if (EtagFactory.matchesIfNoneMatch(inm, etag)) {
    return ResponseEntity.status(304).eTag(etag).build();
  }
  return ResponseEntity.ok().eTag(etag).body(mapper.toResponse(u));
}
```

### 4) Pair with `Last-Modified`

```java
var lm = DateTimeFormatters.rfc1123(user.getUpdatedAt());
return ResponseEntity.ok()
  .eTag(EtagFactory.weak(user.getUpdatedAt()))
  .header("Last-Modified", lm)
  .body(dto);
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/http/EtagFactoryTest.java
package com.example.app.web.support.http;

import org.junit.jupiter.api.Test;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class EtagFactoryTest {

  @Test
  void strongFromBytesIsQuoted() {
    var e = EtagFactory.strong("hello".getBytes(StandardCharsets.UTF_8));
    assertThat(e).startsWith("\"").endsWith("\"");
  }

  @Test
  void weakFromVersionIsWeakQuoted() {
    var e = EtagFactory.weak(42L);
    assertThat(e).startsWith("W/\"").endsWith("\"");
  }

  @Test
  void matchesIfNoneMatchSupportsStarAndLists() {
    var etag = EtagFactory.weak(123);
    assertThat(EtagFactory.matchesIfNoneMatch("*", etag)).isTrue();
    assertThat(EtagFactory.matchesIfNoneMatch("\"nope\", " + etag, etag)).isTrue();
  }

  @Test
  void weakComparisonIgnoresStrength() {
    var strong = EtagFactory.strong("abc");
    var weak   = EtagFactory.toWeak(strong);
    assertThat(EtagFactory.matchesIfNoneMatch(weak, strong)).isTrue();
  }

  @Test
  void strongFromUuidStable() {
    var id = UUID.fromString("00000000-0000-0000-0000-000000000001");
    var e1 = EtagFactory.strong(id);
    var e2 = EtagFactory.strong(id);
    assertThat(e1).isEqualTo(e2);
  }

  @Test
  void weakFromInstantChangesWithTime() {
    var a = EtagFactory.weak(Instant.parse("2025-11-09T10:00:00Z"));
    var b = EtagFactory.weak(Instant.parse("2025-11-09T10:00:01Z"));
    assertThat(a).isNotEqualTo(b);
  }
}
```

---

## Gotchas & guardrails

* **Strong vs weak:** use **strong** only when your outgoing bytes are canonical and stable. Use **weak** for version/`updatedAt`-based equivalence.
* **Canonicalization is your job.** If you serialize JSON, ensure stable ordering/whitespace if you want strong tags to match.
* **Quotes matter.** ETag header values must be quoted; the factory returns **quoted** values already.
* **If-None-Match uses weak comparison** by spec; `W/"x"` matches `"x"`. The helper reflects that.
* **Don’t leak secrets in tags.** They’re opaque to clients, but visible—don’t encode PII/IDs directly unless hashed.

---

## Minimal checklist

* Strong tags from bytes/strings when payload is canonical
* Weak tags from version/Instant
* `matchesIfNoneMatch` to drive 304 responses
* Tests for quoting, weak/strong behavior, star/list handling

---

## Next helper

Finish the **http** cluster with **`ContentDispositionSafe`** for filename-safe downloads (`attachment; filename="..."`; `filename*=UTF-8''...`).
