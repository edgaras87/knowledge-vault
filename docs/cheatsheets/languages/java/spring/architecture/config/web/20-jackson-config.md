---
title: JacksonConfig
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Define a global JSON serialization/deserialization policy for your Spring application using Jackson, ensuring consistent date/time formats, null handling, enum representation, and naming strategies across all controllers.
aliases:
    - Spring Config Web Layer - JacksonConfig Cheatsheet
---

# JacksonConfig — global JSON policy

Define JSON once for the whole app: date/time format, null handling, enums, naming strategy, number precision, and custom (de)serializers. Keep controllers boring.

---

## Where it lives

* **Class:** `com.example.app.config.web.JacksonConfig`
* **Used by:** all controllers automatically (Spring MVC/WEBFlux)

---

## Default policy (recommended)

* **Dates/times:** ISO-8601 strings (never timestamps)
* **Time zone:** UTC (don’t “helpfully” shift)
* **Nulls:** don’t emit (`NON_NULL`)
* **Unknown props:** ignore on input (forward compatibility)
* **Enums:** use strings; fail on unknowns (strict)
* **Numbers:** keep `BigDecimal` precise; don’t write as plain unless required
* **Naming:** keep **camelCase** (switchable to snake_case if you decide)

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/web/JacksonConfig.java
package com.example.app.config.web;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.json.JsonReadFeature;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;
import com.fasterxml.jackson.datatype.jdk8.Jdk8Module;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilderCustomizer;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;

@Configuration
public class JacksonConfig {

  @Bean
  public Jackson2ObjectMapperBuilderCustomizer jsonCustomizer() {
    return builder -> {
      // Modules
      builder.modules(new JavaTimeModule(), new Jdk8Module(), customModule());

      // Global inclusion
      builder.serializationInclusion(JsonInclude.Include.NON_NULL);

      // Date/time as ISO strings (no numeric timestamps)
      builder.featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
      // Don't auto-shift to context TZ; keep what you wrote
      builder.featuresToDisable(DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE);

      // Input leniency and strictness
      builder.featuresToDisable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES); // ignore extras on input
      builder.featuresToEnable(DeserializationFeature.FAIL_ON_NUMBERS_FOR_ENUMS);   // forbid “2” for enums
      builder.featuresToEnable(MapperFeature.ACCEPT_CASE_INSENSITIVE_ENUMS);        // but accept case-insensitive strings

      // Enums as strings everywhere
      builder.featuresToEnable(SerializationFeature.WRITE_ENUMS_USING_TO_STRING);
      builder.featuresToEnable(DeserializationFeature.READ_ENUMS_USING_TO_STRING);

      // Safer JSON reading options (optional)
      builder.postConfigurer((ObjectMapper om) -> {
        om.enable(JsonReadFeature.ALLOW_UNESCAPED_CONTROL_CHARS.mappedFeature());
        om.getFactory().configure(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN, false);
      });

      // Naming strategy (stick to camelCase; switch to SNAKE_CASE if you choose later)
      // builder.propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    };
  }

  /**
   * Custom bits you want everywhere without sprinkling annotations.
   */
  private SimpleModule customModule() {
    var m = new SimpleModule();

    // Example: BigDecimal -> String to avoid scientific notation leaks (optional)
    // m.addSerializer(BigDecimal.class, ToStringSerializer.instance);

    // Example: Force Instant/OffsetDateTime to specific format (ISO always UTC ‘Z’)
    var instantFmt = DateTimeFormatter.ISO_INSTANT;
    m.addSerializer(Instant.class, new com.fasterxml.jackson.datatype.jsr310.ser.InstantSerializer(
        com.fasterxml.jackson.datatype.jsr310.ser.InstantSerializer.INSTANCE, false, instantFmt));

    var odtFmt = DateTimeFormatter.ISO_OFFSET_DATE_TIME.withZone(ZoneOffset.UTC);
    m.addSerializer(OffsetDateTime.class, new com.fasterxml.jackson.datatype.jsr310.ser.OffsetDateTimeSerializer(odtFmt));

    // Add custom (de)serializers here if you need them (e.g., DataSize as "25MB").
    return m;
  }
}
```

> Spring Boot already auto-registers `JavaTimeModule`, but declaring it here makes your intent explicit and keeps all knobs in one place.

---

## Simple DTO proof

```java
public record UserDetailResponse(
  String id,
  String name,
  Instant createdAt,           // → "2025-11-09T16:21:03Z"
  OffsetDateTime lastLogin     // → "2025-11-09T16:21:03Z"
) {}
```

With `WRITE_DATES_AS_TIMESTAMPS=false`, dates render as ISO strings.

---

## Toggling snake_case (team decision)

If you ever want snake_case for the entire API surface:

```java
builder.propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
```

Prefer a single global rule over mixing annotations per record. If a *single* DTO must differ, use `@JsonProperty` only there.

---

## Handling enums cleanly

* Use meaningful enum names (API contract).
* Keep `WRITE_ENUMS_USING_TO_STRING` + `READ_ENUMS_USING_TO_STRING`.
* If you need custom external labels, give the enum a meaningful `toString()` or add `@JsonValue` method.

```java
public enum Role {
  USER, ADMIN;
  @Override public String toString() { return name().toLowerCase(); } // "user", "admin"
}
```

---

## Overriding per-type (mix-in, no source edits)

When you can’t touch a class, attach a mix-in:

```java
// config/JacksonMixins.java
package com.example.app.config.web;

import com.fasterxml.jackson.annotation.JsonInclude;

public final class JacksonMixins {
  private JacksonMixins() {}
  @JsonInclude(JsonInclude.Include.NON_EMPTY)
  public interface NonEmptyMixin {}
}
```

Register in `customModule()`:

```java
// om.addMixIn(TargetType.class, JacksonMixins.NonEmptyMixin.class);
```

---

## Spring properties that pair nicely

```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
    default-property-inclusion: non_null
```

These are redundant with the customizer but useful in tests / quick overrides.

---

## Quick @JsonTest to lock the policy

```java
// src/test/java/com/example/app/config/web/JacksonPolicyTest.java
package com.example.app.config.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.beans.factory.annotation.Autowired;

import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;

@JsonTest
class JacksonPolicyTest {

  @Autowired ObjectMapper om;

  @Test
  void datesAreIsoStrings() throws Exception {
    var json = om.writeValueAsString(new Dto(Instant.parse("2025-11-09T12:34:56Z")));
    assertThat(json).contains("2025-11-09T12:34:56Z").doesNotContain("timestamp");
  }

  record Dto(Instant at) {}
}
```

---

## Custom (de)serializer examples (optional)

### DataSize as “25MB”

```java
// DataSizeSerializer.java
package com.example.app.config.web;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.ser.std.StdSerializer;
import org.springframework.util.unit.DataSize;

import java.io.IOException;

public class DataSizeSerializer extends StdSerializer<DataSize> {
  public DataSizeSerializer() { super(DataSize.class); }
  @Override
  public void serialize(DataSize v, JsonGenerator gen, SerializerProvider sp) throws IOException {
    gen.writeString(v.toMegabytes() + "MB"); // or your own humanizer
  }
}
```

Register it inside `customModule()`:

```java
m.addSerializer(org.springframework.util.unit.DataSize.class, new DataSizeSerializer());
```

> Keep JSON units stable across the API; don’t switch between bytes/MB/GB randomly.

---

## Practical rules of thumb

* Don’t expose JVM-specific types like `Path` or `File` in public JSON; convert to string URIs if needed.
* Keep **one global rule** (camel or snake) to avoid client pain.
* Prefer **immutable DTO records** with explicit field names over auto-mapped entities.
* Avoid timestamps; **always ISO-8601**.
* When you must diverge on a single field, use `@JsonFormat` or `@JsonProperty` locally—**not** another global rule.

---

## Checklist

* `JacksonConfig` registered and tests passing
* ISO-8601 dates, UTC, no timestamps
* Nulls omitted, unknown input tolerated
* Enums as strings, strict on unknown values
* Custom module for one-off types (optional)
* No per-controller JSON tweaks lurking around
