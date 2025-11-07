---
title: Troubleshooting
date: 2025-11-07
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for troubleshooting configuration properties in Spring Boot applications, covering common issues, diagnostic techniques, and solutions for binding failures, missing values, profile mismatches, and more.
aliases:
  - Spring Properties - Troubleshooting Properties Cheatsheet
---

# Troubleshooting Properties — Cheatsheet

Misconfiguration usually shows up at startup:  
binding failures, missing values, wrong types, inconsistent profiles, or unexpected overrides.  
This page collects the common problems and how to diagnose them quickly.

---

# 1. Enable diagnostic logs

These logs reveal where values came from, how they merged, and why binding failed.

```properties
logging.level.org.springframework.boot.context.properties=DEBUG
logging.level.org.springframework.boot.context.config=DEBUG
```

What you see:

- exact file the value came from  
- whether override happened  
- raw → converted type  
- binding failure details  

---

# 2. “Failed to bind” errors

Typical message:

```
Failed to bind properties under 'app.timeout' to java.time.Duration
```

Causes:

- unsupported format (`"5seconds"` instead of `5s`)  
- wrong type (`itemsPerPage: "ten"` instead of `10`)  
- invalid URI  
- wrong YAML indentation  
- property defined as `null`  

Fix:

- correct the YAML  
- verify type in record/class  
- validate nested keys  

---

# 3. Type mismatch errors

Examples:

**Invalid enum:**

```yaml
logging:
  level: INFOO   # typo
```

**Invalid DataSize:**

```yaml
uploads: 100XYZ  # unknown unit
```

**Invalid Duration:**

```yaml
timeout: 20xs
```

Fix:

- cross-check allowed units  
- check capitalization for enums  
- verify YAML indentation  

---

# 4. Missing values (nulls)

Symptoms:

- `NullPointerException` later  
- binding silently sets `null`  
- validation fails with `@NotNull`

Causes:

- YAML key missing  
- incorrect nesting  
- environment variable not set  
- prefix mismatch in `@ConfigurationProperties`

Debug:

- confirm YAML indentation  
- print full path  
- enable debug logs  
- inspect with `/actuator/configprops` (dev only)

---

# 5. Wrong profile loaded

Symptoms:

- “prod” values used during dev  
- overrides not applied  
- profile-specific files ignored

Check:

```properties
logging.level.org.springframework.boot.context.config=DEBUG
```

Then verify:

- what profile is truly active  
- order of `application-*.yml` loading  
- whether `SPRING_PROFILES_ACTIVE` is set in the shell  
- whether a test profile overrides values during tests  

Common mistakes:

- file named `application-production.yml` instead of `application-prod.yml`  
- forgetting `@ActiveProfiles("test")` in tests  
- environment leaking profile values into IDE  

---

# 6. Wrong environment variable mapping

Example:

```
MY_APP_LIMITS_ITEMS_PER_PAGE=10
```

Maps to:

```
my.app.limits.itemsPerPage
```

Common issues:

- forgetting uppercase  
- forgetting underscores  
- using hyphens (`-`) → breaks mapping  
- incorrect nesting (`APP_LIMITS_UPLOADS` vs `APP_LIMITS_UPLOADS_SIZE`)  

Fix:

- verify naming: `PREFIX_NESTED_FIELD`  
- print environment variables seen by the app:

```
/actuator/env
```

---

# 7. Overridden by command-line or IDE run configuration

Symptoms:

- YAML changes do nothing  
- wrong host or timeout  
- unexpected values on startup  

Cause:

- IntelliJ run config silently setting:

```
--spring.profiles.active=dev
--app.host=http://override
```

Fix:

- inspect “Run/Debug Configurations”  
- remove hidden VM options  
- verify built JAR not passing CLI args  

---

# 8. Indentation problems in YAML

Example:

```yaml
app:
  limits:
  itemsPerPage: 20   # wrong indentation
```

Result:

- `limits` becomes null  
- `itemsPerPage` ignored  

Check YAML with a linter or re-indent carefully.

---

# 9. Duplicate definitions coming from multiple sources

Example:

- `application.yml` sets `app.host`  
- `application-dev.yml` overrides  
- local shell sets `APP_HOST`  
- Docker Compose sets `APP_HOST` again  

Final value may not be what you expect.

Debug:

- enable config debug  
- check `/actuator/env`  
- confirm Compose environment overrides  

---

# 10. Secrets missing at runtime

Symptoms:

- startup failure  
- `IllegalArgumentException` on binding  
- null passwords  

Check:

- environment variables set correctly  
- Docker Compose environment section correct  
- secrets not typo’d (`JWT_SECRETS` vs `JWT_SECRET`)  
- avoid `${JWT_SECRET:}` empty default  

---

# 11. Conditional beans not loading

If a class uses:

```java
@ConditionalOnProperty(prefix="app.feature", name="metrics", havingValue="true")
```

Causes of failure:

- boolean value mismatch (`"True"` is fine, `"TRUE"` is fine, `"yes"` is not)  
- prefix incorrect  
- multiple profiles overriding the value  
- missing value and `matchIfMissing=false`

Debug:

- print final value via `props.feature().metrics()`  
- check `/actuator/env`  
- inspect YAML merge order  

---

# 12. Testing overrides interfering with real config

Symptoms:

- dev tests pass but prod starts with wrong values  
- test profile leaking into regular runs  

Fix:

- ensure `@ActiveProfiles` used only in tests  
- isolate test YAML under `src/test/resources`  
- avoid storing real secrets in application.yml  
- ensure CI pipeline sets required env vars  

---

# 13. Startup fails with ambiguous binding paths

Example:

```yaml
app:
  limits:
    itemsPerPage: 20
    itemsperpage: 10   # duplicated key
```

YAML is case-sensitive; binder may pick one unpredictably.

Fix:

- unify naming  
- remove duplicates  

---

# 14. Quick checklist

When configuration “does not work,” check:

1. Is the YAML valid?  
2. Is the profile active?  
3. Did an env var override the value?  
4. Did CLI args override it?  
5. Is the prefix correct in `@ConfigurationProperties`?  
6. Is the type supported (Duration, DataSize, URI)?  
7. Is validation failing and stopping the app?  
8. Are logs enabled for config + binder?  

---

# 15. Quick glossary

- **Binding failure** — conversion from String → Java type failed.  
- **Override** — higher-precedence source wins.  
- **Profile activation** — determines which YAML files load.  
- **Environment variable mapping** — uppercase + underscores → dot notation.  
- **Actuator env** — debug endpoint showing resolved values.  
- **Config data** — YAML/properties loading engine in Spring Boot.
