---
title: ClockConfig
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Define a centralized Clock configuration for your Spring application to ensure consistent time handling across services, tokens, and schedulers by exposing a single Clock bean, with support for different time modes (system, fixed, offset) via properties.
aliases:
    - Spring Config Time Layer - ClockConfig Cheatsheet
---

# ClockConfig — one source of time (UTC)

Expose a single `Clock` bean so services, tokens, and schedulers agree on “now”.
Default is **UTC**; optionally override via properties for tests/replays.

---

## Where it lives

* **Props (optional):** `com.example.app.config.TimeProps`
* **Config:** `com.example.app.config.ClockConfig`
* **Used by:** `JwtService`, expiry policies, audit stamps, schedulers, etc.

---

## YAML we’re binding (optional but recommended)

```yaml
time:
  zone: "UTC"           # any IANA zone, e.g., "Europe/Vilnius"
  mode: system          # system | fixed | offset
  fixed-instant: ""     # e.g., "2025-11-10T10:00:00Z" when mode=fixed
  offset: "PT0S"        # java Duration, e.g., "PT2H" to shift +2h when mode=offset
```

**Profile ideas**

```yaml
# application.yml (default)
time:
  zone: "UTC"
  mode: system

# application-test.yml  (stable tests)
time:
  mode: fixed
  fixed-instant: "2025-01-01T00:00:00Z"
  zone: "UTC"

# application-demo.yml  (pretend it’s tomorrow)
time:
  mode: offset
  offset: "P1D"
  zone: "Europe/Vilnius"
```

---

## Implementation

### 1) Typed properties

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
    @NotBlank String zone,          // IANA zone
    @NotNull Mode mode,             // system | fixed | offset
    String fixedInstant,            // ISO-8601, only when mode=fixed
    @NotNull Duration offset        // only used when mode=offset
) {
  public enum Mode { system, fixed, offset }

  public ZoneId zoneId() { return ZoneId.of(zone); }
  public Instant fixedInstantParsed() {
    if (fixedInstant == null || fixedInstant.isBlank()) return null;
    return Instant.parse(fixedInstant);
  }
}
```

### 2) Clock bean

```java
// src/main/java/com/example/app/config/ClockConfig.java
package com.example.app.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Clock;

@Configuration
public class ClockConfig {

  private final TimeProps props;

  public ClockConfig(TimeProps props) {
    this.props = props;
  }

  @Bean
  public Clock clock() {
    return switch (props.mode()) {
      case system -> Clock.system(props.zoneId());
      case fixed -> {
        var instant = props.fixedInstantParsed();
        if (instant == null)
          throw new IllegalStateException("time.fixed-instant must be set when time.mode=fixed");
        yield Clock.fixed(instant, props.zoneId());
      }
      case offset -> Clock.offset(Clock.system(props.zoneId()), props.offset());
    };
  }
}
```

> If you don’t want property-driven time yet, you can skip `TimeProps` and simply expose `Clock.systemUTC()`; the pattern above keeps you flexible.

---

## Enable binding (if not already scanning `config/`)

```java
// src/main/java/com/example/app/Application.java
package com.example.app;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;

@SpringBootApplication
@ConfigurationPropertiesScan   // picks up TimeProps and others in com.example.app.config
public class Application { }
```

---

## Usage sketch

```java
// src/main/java/com/example/app/security/JwtService.java
package com.example.app.security;

import org.springframework.stereotype.Service;

import java.time.Clock;
import java.time.Instant;
import java.time.temporal.ChronoUnit;

@Service
public class JwtService {
  private final Clock clock;

  public JwtService(Clock clock) { this.clock = clock; }

  public Instant now() { return clock.instant(); }

  public Instant accessExpiryMinutes(int minutes) {
    return now().plus(minutes, ChronoUnit.MINUTES);
  }
}
```

---

## ENV equivalents

```bash
TIME_ZONE=UTC
TIME_MODE=system
# when fixed:
TIME_FIXED_INSTANT=2025-01-01T00:00:00Z
# when offset:
TIME_OFFSET=PT2H
```

---

## Guardrails

* **One `Clock` for everything.** Don’t mix `Instant.now()` with injected `Clock`; inject `Clock` everywhere.
* **Fixed mode requires `fixed-instant`.** Constructor/bean throws if missing.
* **Zones are IANA.** Use `Europe/Vilnius`, not legacy three-letter codes.
* **Profiles, not code edits.** Freeze or offset time per environment with properties.

---

## Tiny tests

```java
// src/test/java/com/example/app/config/ClockConfigTest.java
package com.example.app.config;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneId;

import static org.assertj.core.api.Assertions.assertThat;

class ClockConfigTest {

  private ApplicationContextRunner run(String... p) {
    return new ApplicationContextRunner()
        .withUserConfiguration(Bootstrap.class)
        .withPropertyValues(p);
  }

  @Test
  void fixedClockIsStable() {
    run(
      "time.mode=fixed",
      "time.fixed-instant=2025-01-01T00:00:00Z",
      "time.zone=UTC",
      "time.offset=PT0S"
    ).run(ctx -> {
      var clock = ctx.getBean(Clock.class);
      assertThat(clock.instant()).isEqualTo(Instant.parse("2025-01-01T00:00:00Z"));
      assertThat(clock.getZone()).isEqualTo(ZoneId.of("UTC"));
    });
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(TimeProps.class)
  static class Bootstrap {
    @org.springframework.context.annotation.Bean ClockConfig config(TimeProps p){ return new ClockConfig(p); }
  }
}
```

---

## Checklist

* `TimeProps` bound (`time.mode`, `time.zone`, etc.)
* Single `Clock` bean available across app
* Tests for `fixed`/`offset` modes
* Teams call `Clock`/`Instant` via injection, not `System.currentTimeMillis()`
