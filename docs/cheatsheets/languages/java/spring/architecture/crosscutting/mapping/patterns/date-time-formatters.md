---
title: DateTimeFormatters
date: 2025-11-09
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive guide to implementing DateTimeFormatters in a Java Spring application, providing web-friendly date/time formatters for consistent API responses and HTTP headers.
aliases:
    - Spring Web Support Mapping Layer - DateTimeFormatters Cheatsheet
---

# DateTimeFormatters — Web-friendly time output

Purpose: give your mappers/controllers a tiny, **predictable** set of date/time formats for responses and headers.  
It’s a pure helper (no servlet context), using Java’s thread-safe `DateTimeFormatter`.

- **DTOs:** use ISO 8601 (UTC) by default
- **Headers:** use RFC-1123 (e.g., `Last-Modified`, `Date`)
- **Customization:** choose a `ZoneId` (e.g., user locale) without mutating global state

> Keep storage and internal logic in UTC. Format late, at the web edge.

---

## Where it lives

- **Class:** `com.example.app.web.support.mapping.DateTimeFormatters`
- **Used by:** web mappers (`Instant → String`), `CacheControlPolicy`/`EtagFactory`, controllers
- **Optional in context:** you may pass an instance inside `WebMappingContext` if convenient

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/web/support/mapping/DateTimeFormatters.java
package com.example.app.web.support.mapping;

import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.Locale;
import java.util.Objects;

/**
 * Opinionated, thread-safe datetime format helpers for web output.
 * - ISO-8601 for JSON fields (UTC by default)
 * - RFC-1123 for HTTP date headers
 * - Zone-aware variants without global state
 */
public final class DateTimeFormatters {

  private DateTimeFormatters() {}

  /** ISO-8601 Instant in UTC, e.g., 2025-11-09T08:15:30Z */
  public static String isoInstantUTC(Instant instant) {
    Objects.requireNonNull(instant, "instant");
    return DateTimeFormatter.ISO_INSTANT.format(instant);
    // Same as OffsetDateTime.ofInstant(instant, ZoneOffset.UTC).format(DateTimeFormatter.ISO_INSTANT)
  }

  /** ISO-8601 OffsetDateTime with a given zone, e.g., 2025-11-09T10:15:30+02:00 */
  public static String isoOffset(Instant instant, ZoneId zone) {
    Objects.requireNonNull(instant, "instant");
    Objects.requireNonNull(zone, "zone");
    return DateTimeFormatter.ISO_OFFSET_DATE_TIME.format(instant.atOffset(zone.getRules().getOffset(instant)));
  }

  /** ISO-8601 LocalDate with a given zone (date component only). */
  public static String isoLocalDate(Instant instant, ZoneId zone) {
    Objects.requireNonNull(instant, "instant");
    Objects.requireNonNull(zone, "zone");
    return DateTimeFormatter.ISO_LOCAL_DATE.format(instant.atZone(zone).toLocalDate());
  }

  /** ISO-8601 LocalDateTime with a given zone (no offset in the string). */
  public static String isoLocalDateTime(Instant instant, ZoneId zone) {
    Objects.requireNonNull(instant, "instant");
    Objects.requireNonNull(zone, "zone");
    return DateTimeFormatter.ISO_LOCAL_DATE_TIME.format(instant.atZone(zone).toLocalDateTime());
  }

  /** RFC-1123 for HTTP headers, e.g., Sun, 09 Nov 2025 08:15:30 GMT (always GMT). */
  public static String rfc1123(Instant instant) {
    Objects.requireNonNull(instant, "instant");
    return DateTimeFormatter.RFC_1123_DATE_TIME.format(instant.atOffset(ZoneOffset.UTC));
  }

  /** Friendly short date for humans (rarely used in APIs; pass Locale explicitly). */
  public static String shortDate(Instant instant, ZoneId zone, Locale locale) {
    Objects.requireNonNull(instant, "instant");
    Objects.requireNonNull(zone, "zone");
    Objects.requireNonNull(locale, "locale");
    var fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd", locale);
    return fmt.format(instant.atZone(zone));
  }

  /** Utility: zone from offset minutes (e.g., +120 -> Europe/Vilnius equivalent offset at given instant). */
  public static ZoneId zoneFromOffsetMinutes(int minutes) {
    return ZoneOffset.ofTotalSeconds(minutes * 60);
  }
}
```

---

## Typical use in a mapper

```java
@Mapper(componentModel = "spring")
public interface OrderWebMapper {

  @Mapping(target = "createdAt", expression = "java(com.example.app.web.support.mapping.DateTimeFormatters.isoInstantUTC(src.createdAt()))")
  @Mapping(target = "updatedAt", expression = "java(com.example.app.web.support.mapping.DateTimeFormatters.isoOffset(src.updatedAt(), java.time.ZoneId.of(\"Europe/Vilnius\")))")
  OrderResponse toResponse(Order src);
}
```

**DTO example**

```java
public record OrderResponse(
  String id,
  String createdAt,  // ISO-8601 string
  String updatedAt   // ISO-8601 string with offset
) {}
```

---

## With `WebMappingContext` (optional)

If you often format in the user’s zone/locale, add zone/locale to the context and call:

```java
var s = DateTimeFormatters.isoOffset(entity.updatedAt(), ZoneId.of("Europe/Vilnius"));
```

Or extend your context with a small helper:

```java
public String isoOffset(Instant i) { return DateTimeFormatters.isoOffset(i, zoneFromLocale(locale())); }
```

---

## HTTP headers

```java
// In a controller or http helper
var lastModified = DateTimeFormatters.rfc1123(entity.updatedAt());
return ResponseEntity.ok()
    .header("Last-Modified", lastModified)
    .body(dto);
```

---

## Tests (JUnit + AssertJ)

```java
// src/test/java/com/example/app/web/support/mapping/DateTimeFormattersTest.java
package com.example.app.web.support.mapping;

import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.ZoneId;

import static org.assertj.core.api.Assertions.assertThat;

class DateTimeFormattersTest {

  @Test
  void isoInstantUtcFormatsZ() {
    var i = Instant.parse("2025-11-09T08:15:30Z");
    assertThat(DateTimeFormatters.isoInstantUTC(i)).isEqualTo("2025-11-09T08:15:30Z");
  }

  @Test
  void isoOffsetRespectsZone() {
    var i = Instant.parse("2025-11-09T08:15:30Z");
    var s = DateTimeFormatters.isoOffset(i, ZoneId.of("Europe/Vilnius")); // UTC+2 in November 2025
    assertThat(s).isEqualTo("2025-11-09T10:15:30+02:00");
  }

  @Test
  void rfc1123IsGmt() {
    var i = Instant.parse("2025-11-09T08:15:30Z");
    assertThat(DateTimeFormatters.rfc1123(i)).isEqualTo("Sun, 09 Nov 2025 08:15:30 GMT");
  }
}
```

---

## Gotchas & guardrails

* **Pick one JSON format** for timestamps. ISO-8601 strings are the least surprising.
* **UTC for storage**, convert for display only when you must.
* **Don’t leak locale into machine fields.** Use locale for human text, not for machine timestamps.
* **RFC-1123 is GMT only.** That’s per HTTP spec; don’t put local timezones in headers.

---

## Minimal checklist

* ISO UTC output for DTOs
* RFC-1123 for headers
* Zone-aware helpers when needed
* Tests for format strings and offsets

