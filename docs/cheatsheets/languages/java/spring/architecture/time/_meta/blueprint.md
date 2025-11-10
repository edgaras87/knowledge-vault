---
title: Time Future Blueprint
date: 2025-11-10 
summary: Blueprint for organizing time-related APIs and configuration in a Spring application, separating concerns between time abstractions and their wiring.
---


# Blueprint: Organizing Time APIs and Config in a Spring App

When building a Spring application that deals with time, it's crucial to organize your code in a way that separates concerns clearly. This blueprint outlines a clean structure for managing time-related APIs and their configuration.

Give **time** its own tiny namespace for the *APIs* (qualifier, facade, providers), and keep the *wiring & externalized settings* under `config/`. That split keeps responsibilities clean and avoids “config soup.”

Here’s a layout that plays perfectly and avoids collisions:

```text
src/main/java/com/example/app
├─ config/                         # externalized settings + wiring beans
│  ├─ AppProps.java
│  ├─ JacksonConfig.java
│  ├─ MessageSourceConfig.java
│  ├─ WebConfig.java
│  ├─ CorsProps.java
│  ├─ SecurityProps.java
│  ├─ PasswordPolicyProps.java
│  ├─ StorageProps.java
│  ├─ MailProps.java
│  ├─ TimeProps.java              # ← time profile (zone, mode=fixed/offset/system)
│  ├─ ClockConfig.java            # ← exposes single Clock bean
│  ├─ TimeBeansConfig.java        # ← @Now suppliers (Instant/ZonedDate/LocalDate)
│  └─ TimeFacadeConfig.java       # ← binds Clock → TimeFacade
│
├─ time/                           # tiny, reusable *API* for time
│  ├─ Now.java                     # ← @Qualifier
│  ├─ TimeFacade.java              # ← interface
│  ├─ DefaultTimeFacade.java       # ← impl (pure Java, no Spring annotations)
│  ├─ InstantProvider.java         # ← live provider variant backed by Clock
│  ├─ ZonedNowProvider.java
│  └─ TodayProvider.java
│
├─ web/
│  └─ ...                          # controllers, web/support/*
├─ application/                    # (or service/) use-cases consume Clock/TimeFacade
├─ domain/                         # pure domain; inject Clock/TimeFacade only if needed
└─ infra/                          # adapters
```

### Why this split?

* **`config/` = wiring & policies**
  Anything that binds properties, chooses modes (fixed/offset), or creates Spring beans belongs here. It’s the dial panel.

* **`time/` = neutral, testable API**
  `Now`, `TimeFacade`, and providers are tiny abstractions you can reuse in any layer without dragging Spring or property logic into your domain/app code. They depend only on `java.time` (and `Clock` at construction), which keeps tests trivial.

### Practical rules

* Inject **`Clock`** only in configuration and adapter-style classes.
  In services/use-cases, inject **`TimeFacade`** or `@Now` providers. That keeps time math in one place and reads well.

* Domain layer: prefer to stay framework-agnostic. If a domain service genuinely needs the current time, inject **`TimeFacade`** (constructor param) at the boundary, not `Instant.now()`.

* Tests: set `time.mode=fixed` in `application-test.yml` and enjoy deterministic time. For unit tests, you can `new DefaultTimeFacade(Clock.fixed(...))` directly without Spring.

### Minimal “move plan” (if you already created files elsewhere)

```bash
# APIs (pure Java)
git mv src/main/java/com/example/app/config/Now.java               src/main/java/com/example/app/time/Now.java
git mv src/main/java/com/example/app/config/TimeFacade.java        src/main/java/com/example/app/time/TimeFacade.java
git mv src/main/java/com/example/app/config/DefaultTimeFacade.java src/main/java/com/example/app/time/DefaultTimeFacade.java
git mv src/main/java/com/example/app/config/InstantProvider.java   src/main/java/com/example/app/time/InstantProvider.java
git mv src/main/java/com/example/app/config/ZonedNowProvider.java  src/main/java/com/example/app/time/ZonedNowProvider.java
git mv src/main/java/com/example/app/config/TodayProvider.java     src/main/java/com/example/app/time/TodayProvider.java

# Wiring & props stay under config/
# (TimeProps, ClockConfig, TimeBeansConfig, TimeFacadeConfig already in config/)
```

### TL;DR placement guide

* **Goes in `config/`:** `TimeProps`, `ClockConfig`, `TimeBeansConfig`, `TimeFacadeConfig`
* **Goes in `time/`:** `Now`, `TimeFacade`, `DefaultTimeFacade`, `InstantProvider`, `ZonedNowProvider`, `TodayProvider`
* **Used from:** `application/` services, `web/` controllers → inject `@Now` or `TimeFacade`

