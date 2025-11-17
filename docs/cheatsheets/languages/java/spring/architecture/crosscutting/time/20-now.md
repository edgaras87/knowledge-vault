---
title: '@Now'
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - architecture
    - cheatsheet
summary: Ergonomic injection of “now” suppliers (Instant, ZonedDateTime) backed by your app-wide Clock, keeping business code free of time plumbing while respecting fixed/offset modes for testing.
aliases:
    - Spring Time Layer - @Now Cheatsheet
---

# `@Now` + Suppliers — ergonomic “now” injection

You’ll inject a **`Supplier<Instant>`** (and optionally `Supplier<ZonedDateTime>`) anywhere you need the current time.
Backed by your single `Clock` bean, so fixed/offset modes just work.

## 1) Qualifier annotation

```java
// src/main/java/com/example/app/time/Now.java
package com.example.app.time;

import org.springframework.beans.factory.annotation.Qualifier;

import java.lang.annotation.*;

@Qualifier
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Now {}
```

## 2) Beans that adapt `Clock` → suppliers

```java
// src/main/java/com/example/app/config/TimeBeansConfig.java
package com.example.app.config;

import com.example.app.time.Now;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.*;
import java.util.function.Supplier;

@Configuration
public class TimeBeansConfig {

  /** Supplier of UTC instants (backed by your Clock). */
  @Bean
  @Now
  public Supplier<Instant> nowInstant(Clock clock) {
    return clock::instant;
  }

  /** Supplier of ZonedDateTime using the Clock’s zone (UTC or your configured zone). */
  @Bean
  @Now
  public Supplier<ZonedDateTime> nowZoned(Clock clock) {
    return () -> ZonedDateTime.now(clock);
  }

  /** Optional: LocalDate in clock’s zone. */
  @Bean
  @Now
  public Supplier<LocalDate> today(Clock clock) {
    return () -> LocalDate.now(clock);
  }
}
```

> Same `@Now` qualifier on different types is fine — Spring disambiguates by **type + qualifier**.

## 3) Using it in services/controllers

```java
// src/main/java/com/example/app/security/JwtService.java
package com.example.app.security;

import com.example.app.time.Now;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.function.Supplier;

@Service
public class JwtService {
  private final Supplier<Instant> now;

  public JwtService(@Now Supplier<Instant> now) {
    this.now = now;
  }

  public Instant issuedAt() { return now.get(); }

  public Instant expiresInMinutes(int minutes) {
    return now.get().plus(minutes, ChronoUnit.MINUTES);
  }
}
```

```java
// src/main/java/com/example/app/web/ping/PingController.java
package com.example.app.web.ping;

import com.example.app.time.Now;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.ZonedDateTime;
import java.util.Map;
import java.util.function.Supplier;

@RestController
public class PingController {

  private final Supplier<ZonedDateTime> now;

  public PingController(@Now Supplier<ZonedDateTime> now) {
    this.now = now;
  }

  @GetMapping("/api/ping")
  public Map<String, Object> ping() {
    return Map.of("status", "ok", "time", now.get().toString());
  }
}
```

## 4) Testing is a breeze

Freeze time by overriding the **Clock** (your suppliers follow automatically).

```java
// src/test/java/com/example/app/security/JwtServiceTest.java
package com.example.app.security;

import com.example.app.config.TimeBeansConfig;
import com.example.app.time.Now;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;

class JwtServiceTest {

  @Test
  void usesFixedClock() {
    var fixed = Instant.parse("2025-01-01T00:00:00Z");
    new ApplicationContextRunner()
        .withUserConfiguration(TestCfg.class)
        .run(ctx -> {
          var svc = ctx.getBean(JwtService.class);
          assertThat(svc.issuedAt()).isEqualTo(fixed);
        });
  }

  @Import({TimeBeansConfig.class, JwtService.class})
  static class TestCfg {
    @Bean Clock clock() { return Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC); }
  }
}
```

## 5) Why this pattern?

* Keeps **business code** free of time plumbing.
* Respects your **`ClockConfig`** (`system` / `fixed` / `offset`) with zero changes.
* Makes tests deterministic: swap the **Clock**, not every call site.
* `Supplier<T>` is minimal surface area — no extra utils needed.

## 6) Optional niceties (add if you want)

* Add `@Qualifier("nowInstant")` instead of `@Now` if you prefer built-in qualifiers.
* Provide `Supplier<Long>` for epoch seconds if some clients insist.
* A tiny `TimeFacade` with richer helpers (e.g., `nowUtc()`, `today()`, `after(Duration)`) can sit on top of the suppliers if you want a more domain-y API.

---

### Checklist

* `@Now` qualifier compiled under `com.example.app.time`
* `TimeBeansConfig` creates suppliers from the single `Clock`
* Services inject `@Now Supplier<Instant>` (or `ZonedDateTime`)
* Tests override `Clock` to freeze/shift time
