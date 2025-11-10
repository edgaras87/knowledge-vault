---
title: Default TimeFacade
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - architecture
    - cheatsheet
summary: Concrete TimeFacade implementation backed by the app-wide Clock, providing ergonomic methods for current time, date math, conversions, and timing, ensuring consistent and testable time handling across your services.
aliases:
    - Spring Time Layer - Default TimeFacade Cheatsheet
---

# Default TimeFacade Implementation

Here’s the concrete implementation wired to the app-wide `Clock`. Drop it under `src/main/java/com/example/app/time/DefaultTimeFacade.java`.

```java
// src/main/java/com/example/app/time/DefaultTimeFacade.java
package com.example.app.time;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.*;
import java.util.Objects;
import java.util.function.Supplier;

/**
 * Default TimeFacade backed by the application Clock.
 * Deterministic in tests when Clock is fixed/offset via TimeProps.
 */
public class DefaultTimeFacade implements TimeFacade {
  private static final Logger log = LoggerFactory.getLogger(DefaultTimeFacade.class);

  private final Clock clock;

  public DefaultTimeFacade(Clock clock) {
    this.clock = Objects.requireNonNull(clock, "clock");
  }

  // ---- Instants & dates -----------------------------------------------------

  @Override public Instant now()            { return clock.instant(); }
  @Override public ZonedDateTime nowZoned() { return ZonedDateTime.now(clock); }
  @Override public LocalDate today()        { return LocalDate.now(clock); }

  // ---- Conversions ----------------------------------------------------------

  @Override
  public ZonedDateTime atZone(ZoneId zone, Instant instant) {
    return instant.atZone(Objects.requireNonNull(zone, "zone"));
  }

  @Override
  public Instant fromEpochSeconds(long epochSeconds) {
    return Instant.ofEpochSecond(epochSeconds);
  }

  @Override
  public long toEpochSeconds(Instant instant) {
    return Objects.requireNonNull(instant, "instant").getEpochSecond();
  }

  // ---- Math helpers ---------------------------------------------------------

  @Override public Instant after(Duration d)  { return now().plus(Objects.requireNonNull(d, "duration")); }
  @Override public Instant before(Duration d) { return now().minus(Objects.requireNonNull(d, "duration")); }

  @Override public boolean isPast(Instant t)   { return Objects.requireNonNull(t, "instant").isBefore(now()); }
  @Override public boolean isFuture(Instant t) { return Objects.requireNonNull(t, "instant").isAfter(now()); }

  // ---- Measuring ------------------------------------------------------------

  @Override
  public <T> T measure(String label, Supplier<T> work) {
    long start = System.nanoTime();
    try {
      return work.get();
    } finally {
      long end = System.nanoTime();
      log.debug("time.measure {} took {} ms", label, (end - start) / 1_000_000.0d);
    }
  }

  @Override
  public void measure(String label, Runnable work) {
    measure(label, () -> { work.run(); return null; });
  }

  // ---- Stopwatch ------------------------------------------------------------

  @Override
  public Stopwatch stopwatch() {
    final long started = System.nanoTime();
    return () -> Duration.ofNanos(System.nanoTime() - started);
  }
}
```

If you haven’t yet, expose it as a Spring bean:

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
