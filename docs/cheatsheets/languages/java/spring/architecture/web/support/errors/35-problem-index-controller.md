---
title: ProblemIndexController
date: 2025-11-10
tags: 
    - java
    - spring
    - spring-boot
    - architecture
    - web
    - support
    - errors
    - cheatsheet 
summary: A compact controller that serves a complete index of all registered problem types in both JSON and HTML formats, facilitating easy reference for developers and users alike.
aliases:
    - Spring Web Support Errors Layer - ProblemIndexController Cheatsheet
---

# Problem Index — `GET /problems` (HTML + JSON)

Here’s a compact, copy-paste-ready **Problem Index** that lists every `ProblemKey` with its **slug**, **title**, **default status**, and canonical **type URI**. It serves **JSON** (machines) and **HTML** (humans) from the same endpoint.

## Where it lives

* **Class:** `com.example.app.web.support.errors.ProblemIndexController`
* **Depends on:** `ProblemKey`, `Messages`, `ProblemLinks`, `ProblemTexts` (for default status)

## Controller (production-ready)

```java
// src/main/java/com/example/app/web/support/errors/ProblemIndexController.java
package com.example.app.web.support.errors;

import com.example.app.i18n.Messages;
import com.example.app.i18n.ProblemKey;
import com.example.app.web.support.links.ProblemLinks;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Comparator;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/problems")
public class ProblemIndexController {

  private final Messages msg;
  private final ProblemLinks links;
  private final ProblemTexts texts;

  public ProblemIndexController(Messages msg, ProblemLinks links, ProblemTexts texts) {
    this.msg = msg;
    this.links = links;
    this.texts = texts;
  }

  @GetMapping(produces = { MediaType.APPLICATION_JSON_VALUE, MediaType.TEXT_HTML_VALUE })
  public ResponseEntity<?> index(@RequestHeader(value = "Accept", required = false) String accept) {
    var items = List.of(ProblemKey.values()).stream()
        .sorted(Comparator.comparing(ProblemKey::slug))
        .map(k -> Map.of(
            "key", k.name(),
            "slug", k.slug(),
            "type", links.type(k.slug()).toString(),
            "title", msg.msg(k.titleKey()),
            "defaultStatus", texts.defaultStatusOf(k).value()
        ))
        .toList();

    if (accept != null && accept.contains(MediaType.TEXT_HTML_VALUE)) {
      return ResponseEntity.ok()
          .contentType(MediaType.TEXT_HTML)
          .body(renderHtml(items));
    }
    return ResponseEntity.ok().contentType(MediaType.APPLICATION_JSON).body(Map.of("problems", items));
  }

  // --- trivial HTML (no templating dependency) -------------------------------

  private String renderHtml(List<Map<String, Object>> items) {
    var rows = new StringBuilder();
    for (var it : items) {
      rows.append("""
        <tr>
          <td><code>%s</code></td>
          <td><a href="%s">%s</a></td>
          <td>%s</td>
          <td><span class="badge">%s</span></td>
        </tr>
      """.formatted(
          it.get("key"),
          it.get("type"),
          it.get("slug"),
          it.get("title"),
          it.get("defaultStatus")
      ));
    }

    return """
      <!doctype html>
      <html lang="en"><head>
        <meta charset="utf-8"/>
        <title>Problem Types</title>
        <meta name="viewport" content="width=device-width,initial-scale=1"/>
        <style>
          body{font-family:ui-sans-serif,system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif;margin:2rem}
          table{border-collapse:collapse;width:100%}
          th,td{border:1px solid #e5e7eb;padding:.5rem .75rem;text-align:left}
          th{background:#f9fafb}
          code{background:#f4f4f5;padding:.1rem .25rem;border-radius:.25rem}
          .badge{display:inline-block;border:1px solid #e5e7eb;border-radius:.5rem;padding:.1rem .5rem}
        </style>
      </head><body>
        <h1>Problem Types</h1>
        <p>All registered problem type slugs and their default presentation.</p>
        <table>
          <thead><tr><th>Key</th><th>Slug / Type</th><th>Title</th><th>Status</th></tr></thead>
          <tbody>
            %s
          </tbody>
        </table>
      </body></html>
    """.formatted(rows.toString());
  }
}
```

> Ensure `ProblemTexts` exposes `defaultStatusOf(ProblemKey key)` (we added this earlier).

---

## Tiny add (if you chose “minimal” ProblemKey)

If your enum doesn’t carry status, `ProblemTexts.defaultStatusOf(k)` is the single source of truth. If you embedded status in the enum (enriched pattern), you can use `k.defaultStatus().value()` instead.

---

## Quick smoke tests

```java
// src/test/java/com/example/app/web/support/errors/ProblemIndexControllerTest.java
package com.example.app.web.support.errors;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Import;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(controllers = ProblemIndexController.class)
@Import({ ProblemTexts.class /* + stubs for Messages & ProblemLinks if needed */ })
class ProblemIndexControllerTest {

  @Autowired MockMvc mvc;

  @Test
  void jsonList() throws Exception {
    mvc.perform(get("/problems").header("Accept", "application/json"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.problems").isArray());
  }

  @Test
  void htmlList() throws Exception {
    mvc.perform(get("/problems").header("Accept", "text/html"))
       .andExpect(status().isOk())
       .andExpect(content().contentType("text/html;charset=UTF-8"));
  }
}
```

---

## Checklist

* `ProblemKey`, `Messages`, `ProblemLinks`, `ProblemTexts.defaultStatusOf` present
* `ProblemIndexController` registered under `/problems`
* `GET /problems` returns JSON list and HTML table
* `/problems/{slug}` (from earlier) documents each entry
