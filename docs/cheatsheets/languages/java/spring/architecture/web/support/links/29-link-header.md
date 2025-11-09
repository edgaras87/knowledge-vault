---
title: LinkHeader 
date: 2025-11-08
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing LinkHeader in a Java Spring application, providing an RFC 5988/8288 compliant builder for formatting navigation links into HTTP Link headers.
aliases:
    - Spring Web Support Links Layer - LinkHeader Cheatsheet
---

# LinkHeader — RFC 5988/8288 Builder

Format a set of navigation links (e.g., from `PaginationLinks`) into a single `Link` header:

```
<Link>; rel="self", <Link>; rel="next", <Link>; rel="last"
```

You give it a map `{ rel -> URI }` (ideally a `LinkedHashMap`), optionally add common or per-rel parameters, and it returns a ready-to-send header string.

---

## Where it lives

* **Class:** `com.example.app.web.support.links.LinkHeader`
* **Depends on:** nothing (pure formatter)
* **Used by:** controllers after computing rel → URI maps (often from `PaginationLinks`)

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/links/LinkHeader.java
package com.example.app.web.support.links;

import org.springframework.stereotype.Component;

import java.net.URI;
import java.util.*;
import java.util.stream.Collectors;

@Component
public class LinkHeader {

  // Preferred ordering for pagination rels. Others appended alphabetically.
  private static final List<String> REL_ORDER = List.of("self", "first", "prev", "next", "last");

  /**
   * Build a Link header from rel→URI pairs.
   * If 'rels' is not a LinkedHashMap, entries are ordered using common rel precedence (self,first,prev,next,last) then alphabetically.
   * Null URIs are skipped.
   */
  public String of(Map<String, URI> rels) {
    return of(rels, Map.of(), Map.of());
  }

  /**
   * Build a Link header with common parameters appended to every link, e.g. type="application/json".
   */
  public String of(Map<String, URI> rels, Map<String, String> commonParams) {
    return of(rels, commonParams, Map.of());
  }

  /**
   * Build a Link header with both common parameters and per-rel parameters.
   * perRelParams keys must match rel names; values are key→value parameter maps (e.g., title, hreflang).
   */
  public String of(Map<String, URI> rels, Map<String, String> commonParams, Map<String, Map<String, String>> perRelParams) {
    if (rels == null || rels.isEmpty()) return "";
    var entries = orderedEntries(rels);
    return entries.stream()
        .filter(e -> e.getValue() != null)
        .map(e -> formatOne(e.getKey(), e.getValue(), commonParams, perRelParams.getOrDefault(e.getKey(), Map.of())))
        .collect(Collectors.joining(", "));
  }

  // ---------- internals ----------

  private List<Map.Entry<String, URI>> orderedEntries(Map<String, URI> rels) {
    // If LinkedHashMap, preserve caller order
    if (rels instanceof LinkedHashMap<String, URI>) {
      return new ArrayList<>(rels.entrySet());
    }
    // Else: preferred rel order first, then the rest alphabetically by rel
    var remaining = new TreeMap<String, URI>(String.CASE_INSENSITIVE_ORDER);
    remaining.putAll(rels);

    var ordered = new ArrayList<Map.Entry<String, URI>>();
    for (var rel : REL_ORDER) {
      var uri = remaining.remove(rel);
      if (uri != null) ordered.add(Map.entry(rel, uri));
    }
    ordered.addAll(remaining.entrySet());
    return ordered;
  }

  private String formatOne(String rel, URI uri, Map<String, String> common, Map<String, String> perRel) {
    var sb = new StringBuilder();
    sb.append('<').append(uri.toString()).append('>').append("; rel=\"").append(escapeQuoted(rel)).append('"');
    // merge params: common first, then per-rel (per-rel may override)
    var merged = new LinkedHashMap<String, String>();
    if (common != null) merged.putAll(common);
    if (perRel != null) merged.putAll(perRel);
    for (var e : merged.entrySet()) {
      var k = e.getKey();
      var v = e.getValue();
      if (k == null || k.isBlank() || v == null) continue;
      sb.append("; ").append(token(k)).append("=\"").append(escapeQuoted(v)).append('"');
    }
    return sb.toString();
  }

  // RFC-ish token sanitization: keep simple; strip spaces/quotes/semicolons
  private String token(String s) {
    return s.replaceAll("[\\s\";,]+", "");
  }

  // Minimal quoted-string escaping
  private String escapeQuoted(String s) {
    return s.replace("\\", "\\\\").replace("\"", "\\\"");
  }
}
```

---

## Typical usage

```java
// In a controller after calling PaginationLinks.nav(...)
var rels = paging.nav("users", page, size, totalPages,
    b -> (q == null || q.isBlank()) ? b : b.queryParam("q", q));

var header = linkHeader.of(
    rels,
    Map.of("type", "application/json") // common param for every link (optional)
);

return ResponseEntity.ok()
    .header("Link", header)
    .body(result.items());
```

**Result**

```
</api/v1/users?page=2&size=20&q=A%20B>; rel="self"; type="application/json",
</api/v1/users?page=0&size=20&q=A%20B>; rel="first"; type="application/json",
</api/v1/users?page=1&size=20&q=A%20B>; rel="prev"; type="application/json",
</api/v1/users?page=3&size=20&q=A%20B>; rel="next"; type="application/json",
</api/v1/users?page=9&size=20&q=A%20B>; rel="last"; type="application/json"
```

---

## Per-rel parameters example

```java
var perRel = Map.<String, Map<String, String>>of(
  "next", Map.of("title", "Next page"),
  "prev", Map.of("title", "Previous page")
);
var header = linkHeader.of(rels, Map.of(), perRel);
```

Outputs:

```
<...>; rel="self", <...>; rel="first", <...>; rel="prev"; title="Previous page", ...
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/links/LinkHeaderTest.java
package com.example.app.web.support.links;

import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.LinkedHashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class LinkHeaderTest {

  @Test
  void formatsInPreferredOrderWithCommonParams() {
    var h = new LinkHeader();
    var rels = new LinkedHashMap<String, URI>();
    rels.put("self", URI.create("https://example.com/api/v1/users?page=2&size=50"));
    rels.put("next", URI.create("https://example.com/api/v1/users?page=3&size=50"));
    rels.put("last", URI.create("https://example.com/api/v1/users?page=9&size=50"));

    var header = h.of(rels, Map.of("type", "application/json"));
    assertThat(header).isEqualTo(
      "<https://example.com/api/v1/users?page=2&size=50>; rel=\"self\"; type=\"application/json\", " +
      "<https://example.com/api/v1/users?page=3&size=50>; rel=\"next\"; type=\"application/json\", " +
      "<https://example.com/api/v1/users?page=9&size=50>; rel=\"last\"; type=\"application/json\""
    );
  }

  @Test
  void ordersUnknownRelsAfterKnownOnesAlphabetically() {
    var h = new LinkHeader();
    var rels = Map.of(
      "apple", URI.create("https://e/x?a"),
      "self", URI.create("https://e/x?s"),
      "zulu", URI.create("https://e/x?z")
    );
    var header = h.of(rels);
    assertThat(header).startsWith("<https://e/x?s>; rel=\"self\"");
    assertThat(header).contains("<https://e/x?a>; rel=\"apple\"");
    assertThat(header).endsWith("<https://e/x?z>; rel=\"zulu\"");
  }

  @Test
  void perRelParamsAreAppended() {
    var h = new LinkHeader();
    var rels = new LinkedHashMap<String, URI>();
    rels.put("prev", URI.create("https://e/x?p"));
    var perRel = Map.of("prev", Map.of("title", "Previous page"));
    var header = h.of(rels, Map.of(), perRel);
    assertThat(header).isEqualTo("<https://e/x?p>; rel=\"prev\"; title=\"Previous page\"");
  }
}
```

---

## Gotchas & guardrails

* **Absolute URIs.** Build links with `ApiLinks`/`PaginationLinks`; this class only formats.
* **Stable order.** Pass a `LinkedHashMap` if you care about a specific order; otherwise it applies a sensible default (self→first→prev→next→last→alphabetical).
* **Skip nulls.** Null URIs are ignored; empty maps return `""`.
* **Quoting.** Params are quoted and minimally escaped; avoid placing raw quotes/semicolons in titles.
* **One header.** HTTP allows multiple `Link` headers; this builder emits one comma-separated header string (the common practice).

---

## Minimal checklist

* `of(rels)` prints a single RFC-style header string
* Optional `commonParams` and per-rel params supported
* Null/empty inputs handled gracefully
* Order predictable for pagination rels
* Unit tests cover ordering, parameters, and formatting

---

## Works best with

* `PaginationLinks` to compute the `rels` map
* `ApiLinks` to ensure links live under `/api/v{n}/...`

