---
title: TimeProps
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Centralize timezone and clock mode (system, fixed, offset) into a single validated record, enabling profile-driven time control without code changes.
aliases:
    - Spring Config Props Layer - TimeProps Cheatsheet
---

# TimeProps — profile-driven clock control

Externalize **timezone** and **clock mode** so you can run with system time, a **fixed** instant (tests), or an **offset** clock (demos) without touching code.

---

## YAML we’re binding

```yaml
# application.yml (default)
time:
  zone: "UTC"          # any IANA zone, e.g., "Europe/Vilnius"
  mode: system         # system | fixed | offset
  fixed-instant: ""    # only when mode=fixed, e.g., "2025-01-01T00:00:00Z"
  offset: "PT0S"       # only when mode=offset, java Duration like "PT2H"
```

Profile ideas:

```yaml
# application-test.yml
time:
  mode: fixed
  fixed-instant: "2025-01-01T00:00:00Z"
  zone: "UTC"

# application-demo.yml
time:
  mode: offset
  offset: "P1D"
  zone: "Europe/Vilnius"
```

---

## Where it lives

* **Class:** `com.example.app.config.TimeProps`
* **Used by:** `ClockConfig` (builds the single app `Clock`)

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/TimeProps.java
package com.example.app.config;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;

@Validated
@ConfigurationProperties(prefix = "time")
public record TimeProps(
    @NotBlank String zone,          // IANA zone, e.g., "UTC", "Europe/Vilnius"
    @NotNull Mode mode,             // system | fixed | offset
    String fixedInstant,            // ISO-8601 instant when mode=fixed
    @NotNull Duration offset        // used only when mode=offset (PT…)
) {
  public TimeProps {
    // Basic cross-field validation
    switch (mode) {
      case fixed -> {
        if (fixedInstant == null || fixedInstant.isBlank()) {
          throw new IllegalArgumentException("time.fixed-instant must be set when time.mode=fixed");
        }
      }
      case offset -> {
        if (offset.isZero() || offset.isNegative()) {
          throw new IllegalArgumentException("time.offset must be positive when time.mode=offset");
        }
      }
      case system -> { /* nothing extra */ }
    }
  }

  public enum Mode { system, fixed, offset }

  /** Parsed helpers for ClockConfig / callers */
  public ZoneId zoneId() { return ZoneId.of(zone); }
  public Instant fixedInstantParsed() {
    return (fixedInstant == null || fixedInstant.isBlank()) ? null : Instant.parse(fixedInstant);
  }
}
```

---

## Pairing `ClockConfig` (reference)

```java
// src/main/java/com/example/app/config/ClockConfig.java
package com.example.app.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Clock;

@Configuration
public class ClockConfig {

  private final TimeProps props;

  public ClockConfig(TimeProps props) { this.props = props; }

  @Bean
  public Clock clock() {
    return switch (props.mode()) {
      case system -> Clock.system(props.zoneId());
      case fixed  -> Clock.fixed(props.fixedInstantParsed(), props.zoneId());
      case offset -> Clock.offset(Clock.system(props.zoneId()), props.offset());
    };
  }
}
```

> With this in place, your **`TimeFacade`**, `@Now` providers, JWT expiries, audits—everything—runs off the same clock.

---

## ENV equivalents

```bash
TIME_ZONE=UTC
TIME_MODE=system

# Fixed mode
TIME_FIXED_INSTANT=2025-01-01T00:00:00Z

# Offset mode
TIME_OFFSET=PT2H
```

---

## Tiny test (binding + invariants)

```java
// src/test/java/com/example/app/config/TimePropsTest.java
package com.example.app.config;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TimePropsTest {

  private ApplicationContextRunner run(String... p) {
    return new ApplicationContextRunner()
        .withUserConfiguration(Bootstrap.class)
        .withPropertyValues(p);
  }

  @Test
  void fixedModeRequiresInstant() {
    assertThatThrownBy(() -> run(
      "time.mode=fixed",
      "time.zone=UTC",
      "time.offset=PT0S"
    ).run(c -> c.getBean(TimeProps.class)))
      .hasMessageContaining("fixed-instant");
  }

  @Test
  void offsetMustBePositive() {
    assertThatThrownBy(() -> run(
      "time.mode=offset",
      "time.zone=UTC",
      "time.offset=PT0S"
    ).run(c -> c.getBean(TimeProps.class)))
      .hasMessageContaining("time.offset must be positive");
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(TimeProps.class)
  static class Bootstrap {}
}
```

---

## Checklist

* `@ConfigurationProperties(prefix="time")` bound and validated
* `ClockConfig` builds one `Clock` from `TimeProps`
* Tests cover `fixed`/`offset` invariants
* Services consume **`TimeFacade`** / `@Now` providers—never `Instant.now()` directly


