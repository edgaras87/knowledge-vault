---
title: Loading Configuration
date: 2025-11-03
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
summary: Understand how Spring Boot loads and layers configuration from multiple sources to effectively manage application settings.
aliases:
    - Spring Boot Configuration Loading Explained
---

# How Spring loads and layers configuration (Boot 2.4+ mental model)

---

### 0) Priming the Environment

Before your beans exist, Spring Boot builds an **Environment**—a big key→value map from multiple sources:

1. **Command-line args** `--app.host=https://cli.local`
2. **System properties** `-Dapp.host=https://vm.local`
3. **OS env vars** `APP_HOST=https://env.local`
4. **Config data** (your `application.yml`, `application-*.yml`, and anything you `import`)
5. **Defaults** (hardcoded framework defaults)

They’re stacked in **precedence order** (top wins). That’s why toggling with a CLI flag beats YAML.

---

### 1) Config Data API discovers files

Boot scans for config in a few default places, most importantly:

* `classpath:/application.yml` (your packaged file)
* `classpath:/application-<profile>.yml` (profile file)
* Plus any **additional locations** you specify:

  * CLI/VM: `--spring.config.additional-location=./config/`
  * Inside YAML:

    ```yaml
    spring:
      config:
        import: optional:file:.env.local,optional:file:./secrets.yml
    ```

Those imports are processed **as if** they were part of your config, but with their own place in the order.

---

### 2) Profiles are activated, then profile files are pulled in

Profiles decide which extra files/sections to include.

Ways to activate:

* CLI: `--spring.profiles.active=dev`
* VM: `-Dspring.profiles.active=dev`
* Env: `SPRING_PROFILES_ACTIVE=dev`
* Inside config file (works, but don’t ship prod with this baked in)

In files you can also gate sections with:

```yaml
spring:
  config:
    activate:
      on-profile: dev
```

(Old style `spring.profiles: dev` still works. Newer `spring.config.activate.on-profile` is clearer.)

Once Spring knows active profiles, it merges:

* `application.yml`
* then `application-dev.yml` (and any other active)
* plus imports for each of those

Later sources **override earlier** ones for the same keys.

---

### 3) Placeholders resolve; relaxed names map

Now `${foo.bar}` style placeholders get resolved using the current Environment.
“Relaxed binding” means `app.limits.items-per-page`, `app.limits.itemsPerPage`, and `APP_LIMITS_ITEMS_PER_PAGE` all map to the same property.

This is why placing `APP_HOST` in your shell becomes `app.host` in your Java config class without ceremony.

---

### 4) Binding to your config classes

Only after the Environment is settled do your `@ConfigurationProperties` classes bind:

```java
@ConfigurationProperties("app")
public record AppProps(URI host, Duration timeout, Features features, Limits limits) {}
```

Spring converts **types** for you: `URI`, `Duration`, `DataSize`, enums, `List`, nested objects (your `Features` / `Limits`). If something can’t be converted, you’ll get a helpful “failed to bind” exception at startup—exactly when you want to find out.

---

### 5) Beans are created using those bound values

Now your beans spin up and you can inject `AppProps` anywhere. Your **code config** (under `…/config/`) is the circuitry that reads the **data config** (under `resources/`).

---

## What wins when everything sets the same key?

Think of a “last word wins” stack (highest precedence at the top):

1. `--app.host=https://cli.local` (CLI)
2. `-Dapp.host=https://vm.local` (VM/system property)
3. `APP_HOST=https://env.local` (Env var)
4. `application-dev.yml` (active profile file)
5. `application.yml` (base)

Run an experiment:

```yaml
# application.yml
app: { host: "https://base.local" }

# application-dev.yml
app: { host: "https://dev.local" }
```

Start with:

```
./mvnw spring-boot:run -Dspring.profiles.active=dev -Dapp.host=https://vm.local
```

Result: `https://vm.local` wins.

---

## Smart profile tricks

**Profile groups** (name one logical group that enables several profiles):

```yaml
spring:
  profiles:
    group:
      local: [dev, debug]
```

Run with `-Dspring.profiles.active=local` to enable both `dev` and `debug`.

**Conditional sections** inside a file:

```yaml
spring:
  config:
    activate:
      on-profile: prod

app:
  features:
    signup: false
```

No need for a separate prod file if it’s small—though separate files scale better.

---

## Debugging like a pro

* Start with `--debug` to get a **conditions report** (which configs/beans matched).
* Add Actuator and hit `/actuator/env` to see the **effective property values** and their **origins**.
* If a key won’t bind, check spelling in YAML vs. your record field name (relaxed doesn’t fix different words).

---

## Where to put what (recap, but sharper)

* **`src/main/resources/`**
  Config **data**: `application.yml`, profile files, message bundles, imports like `.env.local`.

* **`src/main/java/.../config/`**
  Config **code**: `@ConfigurationProperties` classes (like `AppProps`), `@Bean` providers (`JacksonConfig`, `CorsConfig`, `SecurityConfig`, etc.).

You don’t put Java classes in `resources/`, and you don’t put YAML in `…/config/`. One is the knob; the other is the wiring.

---

## Common “Why isn’t it working?” pitfalls

* Set `SPRING_PROFILES_ACTIVE= dev` (note the stray space) → profile ignored.
* Wrote `items-per-page` in YAML but used `itemsPerpage` (typo) in Java → binding fails.
* Put `spring.profiles.active` inside `application.yml` and expected CLI to override it—CLI **will** override, but baking profiles into the jar confuses ops; prefer external control.
* Forgot to enable properties scanning:

  * Either `@EnableConfigurationProperties(AppProps.class)` **or** `@ConfigurationPropertiesScan`.

---

## Handy micro-patterns to adopt now

* **Single source of truth for endpoints**
  Put external service URLs in YAML (URIs), bind to `ExternalProps`, and let services inject and use them. No string concatenation gymnastics.

* **Feature toggles live in YAML**
  `app.features.signup`, `app.features.newSearch`. Your controllers/services read toggles and change behavior without code edits.

* **Profile-only overrides are tiny**
  Keep `application.yml` the truth. `application-prod.yml` should be overrides only, not a separate universe.
