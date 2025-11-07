---
title: Properties Cheatsheet Future Blueprint
date: 2025-11-06
summary:
---



# Spring Boot Properties — Blueprint for a Cheatsheet Series

### 0) Index & mental model

* `properties/index.md` — map of the territory: sources, precedence, binders, profiles, testing hooks, common pitfalls. Link out to all subnotes.

### 1) Types & formats (the handy table)

* `properties/types-and-formats.md`
* Scope: built-in conversions: `Duration` (`10s`, `2m`), `DataSize` (`10MB`), `URI`, `Path`, `InetAddress`, `List`, `Map`.
* Include: a one-glance table (value → Java type → sample yml), and gotchas (trailing slashes in `URI`, commas in lists). 


### 2) Reading values (Binding 101)

* `properties/binding.md`
* Scope: `@ConfigurationProperties` (constructor-binding), `@Value`, placeholders, SpEL (when not to), nested records.
* Include: minimal `AppProps` record, `@EnableConfigurationProperties` vs component scanning, where classes live in your layout.

### 3) Validation of properties

* `properties/validation.md`
* Scope: `jakarta.validation` with `@Validated` on `@ConfigurationProperties`, custom constraints, readable error messages.
* Include: quick patterns for `@NotBlank`, `@Min`, regex for hostnames; how to fail fast at startup.

### 4) Profiles & environments

* `properties/profiles.md`
* Scope: `spring.profiles.active`, `application-{profile}.yml`, profile groups, default profile, activating via env vars/CLI.
* Include: “matrix” examples (dev/test/prod), how to override selectively, DO/DON’T for putting secrets in profile files. 

### 5) Property sources & precedence

* `properties/sources-and-precedence.md`
* Scope: file vs env vs CLI args vs `@TestPropertySource` vs `DynamicPropertySource`, `application.yml` vs `.properties`, config data import.
* Include: the exact precedence ladder and “how to diagnose where a value came from”.

### 6) Exposing configuration safely (web edge)

* `properties/exposing-config.md`
* Scope: shaping responses (`*Response` DTOs) without leaking secrets, selective projection, feature flags.
* Include: pattern for a read-only `/api/app` endpoint and a `ProblemLinks` helper (where it lives, why it’s better than hardcoding).

### 7) Feature flags & conditionals

* `properties/feature-flags.md`
* Scope: `@ConditionalOnProperty`, lightweight toggles in `AppProps.feature.*`, profile-based toggles.
* Include: test patterns for toggled code paths.

### 8) Secrets & externalization

* `properties/secrets.md`
* Scope: env vars, `.env` for local only, Vault/1Password/SOPS at a glance, redaction in logs.
* Include: DO/DON’T list and a minimal “local vs CI” setup.

### 9) Reloading & config servers (optional, future)

* `properties/reloading-and-remote.md`
* Scope: devtools vs Spring Cloud Config vs Consul, what’s hot-reloadable and what isn’t.
* Include: blunt guidance on when *not* to add this complexity.

### 10) Testing configuration

* `properties/testing.md`
* Scope: `@ActiveProfiles("test")`, `@TestPropertySource`, `DynamicPropertySource` for ephemeral ports/containers.
* Include: quick templates for JUnit 5 + Spring Boot.

### 11) Troubleshooting binding errors

* `properties/troubleshooting.md`
* Scope: “Failed to bind” patterns, nulls, wrong types, prefix mismatches, list/map binding quirks.
* Include: step-by-step checklist and sample stack traces annotated.

---

## Where these live (your repo)

```
docs/cheatsheets/
└─ spring/
   └─ properties/
      ├─ index.md
      ├─ types-and-formats.md
      ├─ binding.md
      ├─ validation.md
      ├─ profiles.md
      ├─ sources-and-precedence.md
      ├─ exposing-config.md
      ├─ feature-flags.md
      ├─ secrets.md
      ├─ reloading-and-remote.md
      ├─ testing.md
      └─ troubleshooting.md
```

(Keep your **code** in `src/main/java/.../config` for `AppProps`, `ProblemLinks`, etc.; the docs mirror concepts, not packages.)

---

## Opinionated scope boundaries (so notes don’t sprawl)

* **`binding.md`** = how to *get values into Java* (records vs `@Value`, nesting, defaults).
* **`validation.md`** = guarantees on those values (JSR-380/Jakarta constraints).
* **`types-and-formats.md`** = conversion reference (copy-paste table you’ll use daily).
* **`exposing-config.md`** = how to *present* config on the web edge without foot-guns.
* **`feature-flags.md`** = turning behavior on/off cleanly (no if-forest in controllers).

---

## Tiny starter stubs (for consistency)

Put this header at the top of each note so your vault stays queryable:

```markdown
---
title: Properties — Binding
tags: [spring, properties, binding]
summary: How values flow from application.yml/env into strongly-typed records.
aliases: [@ConfigurationProperties, @Value]
---
```

Each note should have:

1. **Why it exists** (2–3 lines).
2. **Minimal pattern** (one code block).
3. **Pitfalls** (bullets).
4. **Links** to sibling themes.

---

## Fast wins you’ll feel immediately

* You’ll stop hunting for “where did this value come from?” — that’s `sources-and-precedence.md`.
* You’ll stop re-Googling `Duration` strings — that’s `types-and-formats.md`.
* You’ll avoid leaking secrets in nice demo endpoints — that’s `exposing-config.md`.
* You’ll make flags boring and safe — `feature-flags.md`.
* You’ll break the “why is binding failing?” loop — `troubleshooting.md`.

Next step: I can turn any one of these into a filled, copy-paste-ready template with examples from your `AppProps` + `ProblemLinks` setup—pick a file name from the list and I’ll ship it.
