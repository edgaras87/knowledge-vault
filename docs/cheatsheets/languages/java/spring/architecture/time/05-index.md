---
title: Content
date: 2025-11-10
tags:  
    - spring
    - spring-boot
    - architecture
    - cheatsheet
summary: Tiny, framework-light helpers for working with time. This folder holds APIs (qualifier, facade, providers) that depend only on java.time. All wiring and property knobs live in config (see ClockConfig, TimeProps, …).
aliases:
    - Spring Time Layer - Content Cheatsheet 
---

# `time/` — Time Utilities Index

Tiny, framework-light helpers for working with time.
This folder holds **APIs** (qualifier, facade, providers) that depend only on `java.time`.
All wiring and property knobs live in `config/` (see `ClockConfig`, `TimeProps`, …).

---

## What lives here

* **`Now`** — qualifier annotation for injecting “now” providers.
* **`TimeFacade`** — small API for “now”, math, conversions, and quick timing.
* **`DefaultTimeFacade`** — implementation backed by the app-wide `Clock`.
* **Providers** — concrete, injectable helpers:

  * `InstantProvider` → `Instant now()`
  * `ZonedNowProvider` → `ZonedDateTime now()`
  * `TodayProvider` → `LocalDate today()`

These give services a clean interface without pulling Spring or property logic into the domain.

---

## File tree

```text
src/main/java/com/example/app/time/
├─ Now.java                  # @Qualifier for time providers
├─ TimeFacade.java           # interface: now(), after(), before(), measure(), …
├─ DefaultTimeFacade.java    # impl backed by java.time.Clock
├─ InstantProvider.java      # live provider: now()
├─ ZonedNowProvider.java     # live provider: now()
└─ TodayProvider.java        # live provider: today()
```

> The **Clock** bean and mode (system/fixed/offset) are defined in `config/`:
> `TimeProps`, `ClockConfig`, `TimeBeansConfig`, `TimeFacadeConfig`.

---

## Quick usage

### Inject a provider

```java
// in a service
import com.example.app.time.Now;
import com.example.app.time.InstantProvider;

@Service
public class Tokens {
  private final InstantProvider now;

  public Tokens(@Now InstantProvider now) { this.now = now; }

  public Instant issuedAt() { return now.now(); }
}
```

### Inject the facade

```java
import com.example.app.time.TimeFacade;

@Service
public class Expiries {
  private final TimeFacade time;

  public Expiries(TimeFacade time) { this.time = time; }

  public Instant accessExpiry(Duration ttl) { return time.after(ttl); }
  public boolean isExpired(Instant t)       { return time.isPast(t); }
}
```

---

## How it’s wired (reference)

* `config/ClockConfig` exposes a single `Clock` (UTC by default; supports fixed/offset via `TimeProps`).
* `config/TimeBeansConfig` creates `@Now` providers from that `Clock`.
* `config/TimeFacadeConfig` exposes `TimeFacade` using `DefaultTimeFacade(clock)`.

No extra setup in this folder; keep it pure and reusable.

---

## Testing pattern

* **Integration tests:** set

  ```yaml
  time:
    mode: fixed
    fixed-instant: "2025-01-01T00:00:00Z"
    zone: "UTC"
  ```

  All services see deterministic time.

* **Unit tests:** construct directly:

  ```java
  var facade = new DefaultTimeFacade(Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC));
  ```

---

## Guidelines

* Inject `TimeFacade` or `@Now` providers in **application/services**; avoid `Instant.now()` in code.
* Keep **domain** code framework-agnostic; pass `TimeFacade` in via constructor if needed.
* Use **UTC** and ISO-8601 everywhere (your `JacksonConfig` already enforces this for JSON).

---

## Checklist

* `time/` contains only API-level helpers (no Spring annotations except `@Qualifier`).
* `config/` holds `Clock` + properties and creates beans.
* Services use `@Now` providers or `TimeFacade`, not `Instant.now()`.
* Tests freeze/offset time via `TimeProps` or `Clock.fixed(...)`.

---

## See also

* `config/TimeProps` — `time.mode`, `time.zone`, `fixed-instant`, `offset`
* `config/ClockConfig` — single `Clock` bean
* `config/TimeBeansConfig` — `@Now` providers
* `config/TimeFacadeConfig` — `TimeFacade` bean
* `config/JacksonConfig` — ISO-8601/UTC JSON dates
