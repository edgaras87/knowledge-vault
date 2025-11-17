---
title: TimeFacade
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - architecture
    - cheatsheet
summary: Define a simple TimeFacade interface and implementation backed by your app-wide Clock, providing ergonomic methods for current time, date math, conversions, and timing, ensuring consistent and testable time handling across your services.
aliases:
    - Spring Time Layer - TimeFacade Cheatsheet
---

# TimeFacade — one tiny API for “now”, dates, and intervals

* Single source of truth: backed by the app-wide `Clock`
* Ergonomic helpers you’ll actually use
* Deterministic in tests (swap the `Clock`, everything follows)

---

## 1) Facade interface

```java
// src/main/java/com/example/app/time/TimeFacade.java
package com.example.app.time;

import java.time.*;
import java.util.function.Supplier;

public interface TimeFacade {

  // Instants & dates
  Instant now();
  ZonedDateTime nowZoned();
  LocalDate today();

  // Conversions
  ZonedDateTime atZone(ZoneId zone, Instant instant);
  Instant fromEpochSeconds(long epochSeconds);
  long toEpochSeconds(Instant instant);

  // Math helpers
  Instant after(Duration d);
  Instant before(Duration d);
  boolean isPast(Instant t);
  boolean isFuture(Instant t);

  // Measuring runtime (inline timing)
  <T> T measure(String label, Supplier<T> work);
  void measure(String label, Runnable work);

  // Stopwatch (manual)
  Stopwatch stopwatch();

  /** Cheap stopwatch handle (nanosecond precision). */
  interface Stopwatch {
    Duration elapsed();
  }
}
```

---

## 2) Default implementation (backed by `Clock`)

```java
// src/main/java/com/example/app/time/DefaultTimeFacade.java
package com.example.app.time;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.*;
import java.util.Objects;
import java.util.function.Supplier;

public class DefaultTimeFacade implements TimeFacade {
  private static final Logger log = LoggerFactory.getLogger(DefaultTimeFacade.class);

  private final Clock clock;

  public DefaultTimeFacade(Clock clock) {
    this.clock = Objects.requireNonNull(clock);
  }

  @Override public Instant now()            { return clock.instant(); }
  @Override public ZonedDateTime nowZoned() { return ZonedDateTime.now(clock); }
  @Override public LocalDate today()        { return LocalDate.now(clock); }

  @Override public ZonedDateTime atZone(ZoneId zone, Instant instant) {
    return instant.atZone(zone);
  }

  @Override public Instant fromEpochSeconds(long epochSeconds) {
    return Instant.ofEpochSecond(epochSeconds);
  }

  @Override public long toEpochSeconds(Instant instant) {
    return instant.getEpochSecond();
  }

  @Override public Instant after(Duration d)  { return now().plus(d); }
  @Override public Instant before(Duration d) { return now().minus(d); }

  @Override public boolean isPast(Instant t)   { return t.isBefore(now()); }
  @Override public boolean isFuture(Instant t) { return t.isAfter(now()); }

  @Override public <T> T measure(String label, Supplier<T> work) {
    long start = System.nanoTime();
    try {
      return work.get();
    } finally {
      long end = System.nanoTime();
      log.debug("time.measure {} took {} ms", label, (end - start) / 1_000_000.0d);
    }
  }

  @Override public void measure(String label, Runnable work) {
    measure(label, () -> { work.run(); return null; });
  }

  @Override public Stopwatch stopwatch() {
    final long started = System.nanoTime();
    return () -> Duration.ofNanos(System.nanoTime() - started);
  }
}
```

---

## 3) Spring wiring

```java
// src/main/java/com/example/app/config/TimeFacadeConfig.java
package com.example.app.config;

import com.example.app.time.DefaultTimeFacade;
import com.example.app.time.TimeFacade;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Clock;

@Configuration
public class TimeFacadeConfig {

  @Bean
  public TimeFacade timeFacade(Clock clock) {
    return new DefaultTimeFacade(clock);
  }
}
```

> Your existing `ClockConfig` (system/fixed/offset) remains the single point of control.

---

## 4) Using it in services

```java
// src/main/java/com/example/app/security/Tokens.java
package com.example.app.security;

import com.example.app.time.TimeFacade;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

@Service
public class Tokens {
  private final TimeFacade time;

  public Tokens(TimeFacade time) {
    this.time = time;
  }

  public Instant issuedAt() { return time.now(); }
  public Instant expiresIn(Duration ttl) { return time.after(ttl); }

  public boolean isExpired(Instant exp) { return time.isPast(exp); }
}
```

---

## 5) Tests are trivial (freeze the Clock)

```java
// src/test/java/com/example/app/time/TimeFacadeTest.java
package com.example.app.time;

import org.junit.jupiter.api.Test;

import java.time.*;

import static org.assertj.core.api.Assertions.assertThat;

class TimeFacadeTest {

  @Test
  void respectsFixedClock() {
    var fixed = Instant.parse("2025-01-02T03:04:05Z");
    var facade = new DefaultTimeFacade(Clock.fixed(fixed, ZoneOffset.UTC));

    assertThat(facade.now()).isEqualTo(fixed);
    assertThat(facade.today()).isEqualTo(LocalDate.of(2025, 1, 2));
    assertThat(facade.after(Duration.ofMinutes(10))).isEqualTo(fixed.plusSeconds(600));
  }

  @Test
  void stopwatchMeasuresElapsed() throws InterruptedException {
    var facade = new DefaultTimeFacade(Clock.systemUTC());
    var sw = facade.stopwatch();
    Thread.sleep(5);
    assertThat(sw.elapsed()).isGreaterThan(Duration.ZERO);
  }
}
```

---

## 6) Optional sugar (add if/when needed)

* `Instant clamp(Instant value, Instant min, Instant max)`
* `boolean within(Instant t, Instant start, Instant end)`
* `Instant roundToSeconds(Instant t)`
* A `UtcTimeFacade` variant that always returns `Z` times (if you ever allow non-UTC `Clock` zones)

---

## 7) Quick checklist

* `TimeFacade` + `DefaultTimeFacade` compiled under `com.example.app.time`
* `TimeFacadeConfig` exposes a singleton bean wired to the app `Clock`
* Services use `TimeFacade` (or `@Now` suppliers) — never `Instant.now()`
* Fixed/offset time fully controlled by `TimeProps` profiles

