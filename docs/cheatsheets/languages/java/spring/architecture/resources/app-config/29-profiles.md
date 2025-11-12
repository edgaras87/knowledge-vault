---
title: Profiles
date: 2025-11-06
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for Spring Boot profiles, detailing how to manage environment-specific configuration using profiles in YAML files and Java code.
aliases:
  - Spring Properties - Profiles Cheatsheet
---


# Spring Profiles — Cheatsheet

Spring profiles control which configuration applies in which environment  
(e.g., `dev`, `test`, `prod`).  
A profile decides which YAML files load, which beans activate, and how values override.  
Profiles are not feature flags; they define *runtime environment shape*.

---

## 1. How profiles select YAML files

Spring loads configuration in this order:

```
application.yml
application-<profile>.yml  # if profile is active
```

Example layout:

```
src/main/resources/
├─ application.yml
├─ application-dev.yml
├─ application-test.yml
└─ application-prod.yml
```

A profile file **only overrides what it redefines**.

**Example**

```yaml
# application.yml
app:
  host: http://localhost:8080
  timeout: 5s
```

```yaml
# application-prod.yml
app:
  host: https://api.example.com
```

`timeout` isn’t redefined in prod → inherited from the base file.

---

## 2. Activating a profile

**Runtime (CLI)**  
```bash
java -jar app.jar --spring.profiles.active=dev
```

**Environment variable**  
```bash
export SPRING_PROFILES_ACTIVE=dev
```

**application.yml** (use for local dev, not for prod)  
```yaml
spring:
  profiles:
    active: dev
```

**Multiple profiles**

```bash
--spring.profiles.active=dev,debug
```

---

## 3. Profile groups

Attach multiple profiles to one name:

```yaml
spring:
  profiles:
    group:
      local: [dev, debug, metrics]
      staging: [prod, metrics]
```

Then:

```
--spring.profiles.active=local
```

Activates: `dev`, `debug`, `metrics`.

---

## 4. Conditional beans with @Profile

```java
@Configuration
@Profile("dev")
public class DevConfig { }
```

Or on a bean:

```java
@Bean
@Profile("prod")
DataSource prodDataSource() { ... }
```

Multiple profiles (OR):

```java
@Profile({"dev", "test"})
```

Require all (AND):

```java
@Profile("dev & debug")
```

---

## 5. Testing with profiles

```java
@SpringBootTest
@ActiveProfiles("test")
class UserServiceTest { }
```

Combine with overrides:

```java
@TestPropertySource(properties = "app.timeout=1ms")
```

---

## 6. DO / DON'T / pitfalls

**DO**
- Use profiles for *environment difference* (dev/test/prod/staging).
- Keep prod config minimal and explicit.
- Use profile grouping when multiple profiles must always travel together.

**DON'T**
- Don’t use profiles to toggle small features → use booleans in properties.
- Don’t rely on `spring.profiles.active` inside code for logic.
- Don’t hardcode defaults in Java when they belong in YAML.

**Pitfalls**
- Loading wrong file because of missing `application.yml` prefix.  
- Missing value inheritance (dev file only overrides, not replaces).  
- `@Profile` on beans breaking autowiring if profile isn’t active.

---

## 7. Practical example (matches your AppProps style)

**application.yml**

```yaml
app:
  host: http://localhost:8080
  limits:
    itemsPerPage: 20
```

**application-dev.yml**

```yaml
app:
  host: http://localhost:8081
  limits:
    itemsPerPage: 5
```

**application-prod.yml**

```yaml
app:
  host: https://api.example.com
```

Usage:

```java
@ConfigurationProperties(prefix = "app")
public record AppProps(URI host, Limits limits) {
  public record Limits(int itemsPerPage) { }
}
```

If `dev` is active, values come from `application.yml` + overrides from `application-dev.yml`.

---

## 8. Quick inspection (Actuator)

If Actuator is enabled:

```
/actuator/env
/actuator/configprops
```

Useful to verify what profile is active and where values originated.

---

## 9. Minimal troubleshooting

- Value missing? Check spelling in YAML and prefix in the record.  
- Wrong file loading? Print active profiles on startup:
  ```
  logging.level.org.springframework.boot.context.config=DEBUG
  ```
- Test using wrong config? Ensure `@ActiveProfiles("test")`.

---

## 10. IntelliJ IDEA (Community) Setup

Organize your YAML files like this:

```
src/main/resources/
├─ application.yml               # defaults (always loaded)
├─ application-local.yml         # overrides for local dev
├─ application-dev.yml           # overrides for dev/staging
└─ application-prod.yml          # overrides for prod
```


You’ll use **Application** run configs.

1. Run → **Edit Configurations…**
2. Select your app, click **Modify options**, enable **VM options**.
3. Duplicate the config for each profile.
4. Set in **VM options** (preferred):

```
App — local : -Dspring.profiles.active=local
App — dev   : -Dspring.profiles.active=dev
App — prod  : -Dspring.profiles.active=prod
```

Alternatively:

* Program arguments → `--spring.profiles.active=dev`
* or Environment variable → `SPRING_PROFILES_ACTIVE=dev`

---

### Verify active profile

```java
@SpringBootApplication
public class ShoppingCartApplication {
  public ShoppingCartApplication(org.springframework.core.env.Environment env) {
    System.out.println(">> Active profiles: " + String.join(",", env.getActiveProfiles()));
  }
  public static void main(String[] args) {
    org.springframework.boot.SpringApplication.run(ShoppingCartApplication.class, args);
  }
}
```

Run your `App — dev` config; the console should print:

```
>> Active profiles: dev
```

---

### Rules of thumb

* Never hard-code `spring.profiles.active` inside `application.yml`.
  → Always choose via run config, env var, or CLI flag.
* Spring loads in order:
  `application.yml` → `application-<profile>.yml` (later wins on conflicts).
* You can activate multiple profiles:
  `-Dspring.profiles.active=dev,metrics`
* Tests use their own: `@ActiveProfiles("test")`.


### TL;DR

| Environment | Command / Config Flag            | YAML File Used          |
| ----------- | -------------------------------- | ----------------------- |
| Local dev   | `-Dspring.profiles.active=local` | `application-local.yml` |
| Team dev    | `-Dspring.profiles.active=dev`   | `application-dev.yml`   |
| Production  | `-Dspring.profiles.active=prod`  | `application-prod.yml`  |




---

This file covers:

- activation
- YAML override rules
- profile groups
- bean conditional activation
- test usage
- pitfalls
- practical example from your structure
- IntelliJ setup