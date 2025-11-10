---
title: Config Future Blueprint
date: 2025-11-06
summary: A blueprint for organizing the configuration layer in a Spring Boot application, focusing on scalability and maintainability.
---

# Config Future Blueprint

Keep `config/` mostly flat until it hurts, then split by **concern**, not by framework. You want just enough drawers to find things in 6 months—no nesting matryoshka.

Here’s a setup that scales without becoming busywork.

## Baseline (start here)

Keep a flat handful of files. This covers 80% of projects.

```
src/main/java/com/example/app/config/
├─ AppProps.java              # @ConfigurationProperties (host, api.version, feature flags)
├─ WebConfig.java             # Jackson, CORS, message converters, locale resolver
├─ MessageSourceConfig.java   # i18n basenames
├─ ClockConfig.java           # Clock bean (UTC)
└─ ClientConfig.java          # shared HTTP client(s) if any
```

Rules:

* One file per cross-cutting concern.
* No business logic, no controllers, no JPA entities.
* If a file passes ~200–300 lines or mixes unrelated knobs, split.

## When to add folders (heuristics)

Create a subfolder **only** when:

* You have **3+ files** in the same concern, or
* One area needs its **own tests**/fixtures, or
* You must isolate **environment differences** (dev vs prod wiring).

If a concern doesn’t meet those, keep it flat.

## Future-proof folders that actually pay off

```
src/main/java/com/example/app/config/
├─ props/                 # many @ConfigurationProperties records
│  ├─ AppProps.java
│  ├─ SecurityProps.java
│  └─ StorageProps.java
├─ web/                   # web runtime knobs (serialization, CORS, locale)
│  ├─ WebConfig.java
│  └─ CorsConfig.java
├─ i18n/                  # message source + resolvers
│  └─ MessageSourceConfig.java
├─ time/                  # time, zone, scheduling
│  └─ ClockConfig.java
├─ clients/               # outbound HTTP clients (beans, interceptors)
│  └─ ClientConfig.java
├─ cache/                 # caches if you add Redis/Caffeine later
│  └─ CacheConfig.java
└─ SecurityProps.java     # small stragglers can stay at root until they grow
```

Notes:

* **props/** works well once you have more than ~3 property records.
* **web/** is only for **web runtime** (Jackson/CORS/Locale). Web-edge helpers still live in `web/support/`.
* **clients/** is only wiring (timeouts, base URLs). Concrete adapters still live under `infra/http/…`.
* **cache/** only if you actually add caching; otherwise skip.

## What *not* to put here

* JPA configuration and repositories → `infra/persistence/**`.
* Feign/WebClient wrappers for specific APIs → `infra/http/**`.
* ProblemDetail translators, link builders → `web/support/**`.
* Security filters with domain logic → usually `web/security/**` (or `security/` root package), not `config/`.

## Naming & content conventions

* Files named by **policy** not framework: `WebConfig`, `MessageSourceConfig`, `ClockConfig`, `CorsConfig`, `ClientConfig`.
* Property records end with `Props`: `AppProps`, `MailProps`, `StorageProps`.
* Keep each class **single-purpose**; prefer adding a new tiny class over stuffing one big “GodConfig”.

## Migration path (when it starts to sprawl)

1. Create the folder (e.g., `config/props/`).
2. Move the related classes.
3. Update imports; package rename is fine—these are config classes, not part of public API.
4. Keep bean names stable if consumers use `@Qualifier` (rare).

## Sanity checklist

* Can a newcomer guess where to tweak Jackson? **`config/web/WebConfig.java`**
* Can you find all `@ConfigurationProperties` quickly? **`config/props/`**
* Do HTTP client timeouts live away from business logic? **`config/clients/`**
* Are there fewer than ~8 files at the root? If not, consider a folder.

## Example “grown-up” layout

```
config/
├─ props/
│  ├─ AppProps.java
│  ├─ SecurityProps.java
│  └─ StorageProps.java
├─ web/
│  ├─ WebConfig.java
│  └─ CorsConfig.java
├─ i18n/
│  └─ MessageSourceConfig.java
├─ time/
│  └─ ClockConfig.java
└─ clients/
   └─ ClientConfig.java

```

Keep it boring and obvious. If you’re unsure whether to add a folder, don’t. You can always promote later when a cluster of files emerges. 
