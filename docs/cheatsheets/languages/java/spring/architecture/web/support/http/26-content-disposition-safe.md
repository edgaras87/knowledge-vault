---
title: ContentDispositionSafe
date: 2025-11-09
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing ContentDispositionSafe in a Java Spring application, providing header-ready Content-Disposition strings for secure and user-friendly file downloads.
aliases:
    - Spring Web Support Http Layer - ContentDispositionSafe Cheatsheet
---

# ContentDispositionSafe — bulletproof filenames for browsers

When serving downloads you want:

- **Correct disposition**: `inline` (view in browser) or `attachment` (force download).
- **Safe filename**: ASCII fallback for old/quirky UAs, **UTF-8** filename via `filename*` (RFC 5987).
- **Sanitization**: no path separators, control chars, or header injection.
- **Extension preserved** so the OS knows which app to open.

This helper returns a **header-ready** `Content-Disposition` string. Zero servlet context required.

---

## Where it lives

- **Class:** `com.example.app.web.support.http.ContentDispositionSafe`
- **Used by:** controllers returning files/exports/reports
- **Pairs with:** `CacheControlPolicy`, `EtagFactory`, `DateTimeFormatters` (for `Last-Modified`)

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/http/ContentDispositionSafe.java
package com.example.app.web.support.http;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.text.Normalizer;
import java.util.Objects;
import java.util.regex.Pattern;

/**
 * Build safe Content-Disposition headers with ASCII fallback and RFC 5987 filename* param.
 * Example:
 *   attachment; filename="report.csv"; filename*=UTF-8''%E2%9C%93-report.csv
 */
public final class ContentDispositionSafe {

  private static final int MAX_ASCII_LEN = 60;   // conservative header-friendly length
  private static final int MAX_UTF8_LEN  = 180;  // for filename* (percent-encoded grows)
  private static final Pattern BAD_CHARS = Pattern.compile("[\\r\\n\\t\\0]"); // header injection / control chars
  private static final Pattern SEP_CHARS = Pattern.compile("[/\\\\]");        // path separators
  private static final Pattern QUOTES    = Pattern.compile("\"");             // must be escaped/replaced

  private ContentDispositionSafe() {}

  // ---------- Public API ----------

  /** Build an 'attachment' header with a safe filename. */
  public static String attachment(String suggestedFileName) {
    return header("attachment", suggestedFileName);
  }

  /** Build an 'inline' header with a safe filename (helps some UAs display tab titles correctly). */
  public static String inline(String suggestedFileName) {
    return header("inline", suggestedFileName);
  }

  /** Core builder: disposition = "attachment" or "inline". */
  public static String header(String disposition, String suggestedFileName) {
    Objects.requireNonNull(disposition, "disposition");
    Objects.requireNonNull(suggestedFileName, "suggestedFileName");

    // 1) sanitize + normalize
    var raw = sanitize(suggestedFileName);
    var ext = extractExtension(raw);
    var base = stripExtension(raw);

    // 2) ASCII fallback (lossy): decompose accents → strip non-ASCII → replace spaces → clamp length
    var asciiBase = toAscii(base);
    asciiBase = replaceUnsafe(asciiBase);
    asciiBase = clamp(asciiBase, MAX_ASCII_LEN - ext.length());

    var ascii = ensureNonEmpty(asciiBase) + ext;

    // 3) UTF-8 primary name (preferred by modern browsers) via RFC 5987
    var utf8Base = clamp(base, MAX_UTF8_LEN - ext.length()); // keep a sane size before encoding blowup
    var utf8 = utf8Base + ext;
    var encoded = rfc5987Encode(utf8);

    // 4) Quote ASCII fallback safely
    var quotedAscii = quote(ascii);

    return disposition + "; " +
        "filename=" + quotedAscii + "; " +
        "filename*=UTF-8''" + encoded;
  }

  // ---------- Sanitization / helpers ----------

  /** Remove CR/LF/TAB/NULL, path separators, and normalize Unicode (NFKD). */
  static String sanitize(String name) {
    var norm = Normalizer.normalize(name, Normalizer.Form.NFKD);
    norm = BAD_CHARS.matcher(norm).replaceAll("");
    norm = SEP_CHARS.matcher(norm).replaceAll("-");
    // Trim surrounding spaces and dots (Windows quirk)
    norm = norm.trim();
    while (norm.startsWith(".")) norm = norm.substring(1);
    return norm.isEmpty() ? "file" : norm;
  }

  static String extractExtension(String name) {
    var i = name.lastIndexOf('.');
    if (i <= 0 || i == name.length() - 1) return "";
    var ext = name.substring(i); // includes dot
    // avoid multiple dots chains like ".tar.gz" — keep whole tail, it's fine:
    return ext;
  }

  static String stripExtension(String name) {
    var i = name.lastIndexOf('.');
    if (i <= 0) return name;
    return name.substring(0, i);
  }

  /** ASCII fold (remove accents), drop non-ASCII, collapse spaces to dashes. */
  static String toAscii(String s) {
    var n = Normalizer.normalize(s, Normalizer.Form.NFKD);
    var sb = new StringBuilder(n.length());
    for (int i = 0; i < n.length(); i++) {
      char c = n.charAt(i);
      if (c <= 0x7F) sb.append(c);
    }
    return sb.toString();
  }

  static String replaceUnsafe(String s) {
    s = s.replace(' ', '-');
    s = s.replaceAll("[^A-Za-z0-9._-]", "-");
    s = s.replaceAll("-{2,}", "-");
    s = s.replaceAll("^[-_.]+|[-_.]+$", ""); // trim noisy punctuation
    return s.isEmpty() ? "file" : s;
  }

  static String clamp(String s, int maxLen) {
    if (maxLen <= 0) return "";
    return s.length() <= maxLen ? s : s.substring(0, maxLen);
  }

  static String ensureNonEmpty(String s) {
    return (s == null || s.isBlank()) ? "file" : s;
  }

  static String quote(String ascii) {
    // RFC: quoted-string with backslash-escaped quotes if needed
    var safe = QUOTES.matcher(ascii).replaceAll("\\\\\"");
    return "\"" + safe + "\"";
  }

  /** RFC 5987 percent-encoding: UTF-8 bytes with reserved set encoded. */
  static String rfc5987Encode(String s) {
    // URLEncoder encodes space as '+', which RFC 5987 forbids -> replace '+' with "%20"
    var enc = URLEncoder.encode(s, StandardCharsets.UTF_8)
        .replace("+", "%20")
        .replace("*", "%2A")
        .replace("%7E", "~"); // keep ~ unencoded
    return enc;
  }
}
```

---

## Usage

```java
@GetMapping("/reports/{id}")
public ResponseEntity<byte[]> download(@PathVariable long id) {
  var report = service.generateCsv(id);           // bytes
  var fileName = "✓-report-" + id + ".csv";       // may include Unicode

  String cd = ContentDispositionSafe.attachment(fileName);
  // e.g.: attachment; filename="report-123.csv"; filename*=UTF-8''%E2%9C%93-report-123.csv

  return ResponseEntity.ok()
      .header("Content-Type", "text/csv; charset=utf-8")
      .header("Content-Disposition", cd)
      .header("Cache-Control", CacheControlPolicy.noCache())
      .eTag(EtagFactory.strong(report))
      .body(report);
}
```

Inline display (PDF in browser):

```java
var cd = ContentDispositionSafe.inline("invoice-2025-11-09.pdf");
return ResponseEntity.ok()
    .header("Content-Type", "application/pdf")
    .header("Content-Disposition", cd)
    .body(pdfBytes);
```

---

## Behavior & examples

Input → Header snippet:

* `✓ report 42.csv` → `filename="report-42.csv"; filename*=UTF-8''%E2%9C%93%20report%2042.csv`
* `../../evil.exe`  → `filename="evil.exe"; filename*=UTF-8''evil.exe`
* `   ....`         → `filename="file"; filename*=UTF-8''file`

Long names are truncated: ASCII fallback is clamped to ~60 chars (extension preserved); UTF-8 name is clamped ~180 chars pre-encoding.

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/http/ContentDispositionSafeTest.java
package com.example.app.web.support.http;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ContentDispositionSafeTest {

  @Test
  void buildsAttachmentWithAsciiAndUtf8() {
    var cd = ContentDispositionSafe.attachment("✓ report 42.csv");
    assertThat(cd).startsWith("attachment; ");
    assertThat(cd).contains("filename=\"report-42.csv\"");
    assertThat(cd).contains("filename*=UTF-8''%E2%9C%93%20report%2042.csv");
  }

  @Test
  void sanitizesPathSeparatorsAndControls() {
    var cd = ContentDispositionSafe.attachment("../evil\nname\t.txt");
    assertThat(cd).contains("filename=\"evilname.txt\"");
  }

  @Test
  void preservesExtensionAndClampsAscii() {
    var longName = "x".repeat(200) + ".tar.gz";
    var cd = ContentDispositionSafe.attachment(longName);
    // ASCII fallback is <= ~60 + ext
    var ascii = cd.substring(cd.indexOf("filename=\"") + 10, cd.indexOf("\"; filename*"));
    assertThat(ascii).endsWith(".gz");
    assertThat(ascii.length()).isLessThanOrEqualTo(65);
  }

  @Test
  void inlineDisposition() {
    var cd = ContentDispositionSafe.inline("file.pdf");
    assertThat(cd).startsWith("inline;");
  }
}
```

---

## Gotchas & guardrails

* **Always send both**: `filename` (ASCII fallback) **and** `filename*` (UTF-8). Modern browsers prefer `filename*`, older ones use `filename`.
* **Never trust input**: sanitize out CR/LF, tabs, NUL, and path separators. This avoids header injection and path traversal surprises.
* **Keep extensions**: MIME type detection and OS associations rely on it.
* **Clamping**: long names can break headers or UAs; clamp early and predictably.
* **Content-Type matters**: don’t rely on extension alone—set correct MIME (`application/pdf`, `text/csv`, etc.).

---

## Minimal checklist

* Use `attachment(...)` for downloads, `inline(...)` for previews
* Provide ASCII `filename=` and UTF-8 `filename*=UTF-8''...`
* Sanitize and clamp names; preserve extension
* Pair with `Cache-Control`/`ETag`/`Last-Modified` for a polished download experience
