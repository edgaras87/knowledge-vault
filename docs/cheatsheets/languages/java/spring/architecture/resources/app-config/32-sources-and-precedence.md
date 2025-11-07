---
title: Source & Precedence
date: 2025-11-06
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for Spring Boot configuration property sources and precedence, detailing how different configuration sources interact and override each other.
aliases:
    - Spring Properties - Source And Precedence Cheatsheet
---



# Sources & Precedence — Cheatsheet

Spring collects configuration from many places: YAML, `.properties`, environment variables, system properties, command-line flags, `@TestPropertySource`, and more.  
When multiple sources define the same key, the highest-precedence value wins.  
This file clarifies the order and how to reason about overrides.

---

# 1. The full precedence ladder (highest first)

1. **Command-line arguments**  
   `--app.host=https://override.example`
2. **Java system properties**  
   `-Dapp.host=https://override.example`
3. **Environment variables**  
   `APP_HOST=https://override.example`
4. **`application-<profile>.yml`** (active profiles)
5. **`application.yml`**
6. **Packaged `application.properties` or `.yml` inside JAR**  
7. **Default values inside code** (constructor defaults, field defaults)

Binding simply receives *the final resolved value*.

---

# 2. Config data loading (Spring Boot 2.4+)

Spring loads configuration in a deterministic sequence:

```
application.yml
application-<profile>.yml
```

If multiple profiles are active:

```
application-dev.yml
application-debug.yml
application-metrics.yml
```

Files are merged in the order profiles are activated.

Example:

```
--spring.profiles.active=dev,debug
```

Load order:

1. `application-dev.yml`
2. `application-debug.yml`

Last one wins on conflicts.

---

# 3. Merging rules

Only keys defined in the override file replace parent values.

Example:

**application.yml**

```yaml
app:
  host: http://localhost
  timeout: 10s
  limits:
    itemsPerPage: 25
    uploads: 10MB
```

**application-prod.yml**

```yaml
app:
  timeout: 5s
```

Result:

```
host = http://localhost
timeout = 5s
limits.itemsPerPage = 25
limits.uploads = 10MB
```

Spring merges; it does not replace whole objects unless explicitly stated.

---

# 4. Environment variables → YAML keys

Environment variables follow these transformations:

```
MY_APP_LIMITS_UPLOADS  →  app.limits.uploads
APP_TIMEOUT            →  app.timeout
```

Rules:
- Uppercase → lowercase  
- `_` → `.`  
- Nested paths allowed

Example:

```bash
export APP_HOST=https://env.local
```

Overrides:

```yaml
app:
  host: http://localhost
```

---

# 5. Command-line arguments

Highest-precedence normal input.

```
java -jar app.jar --app.timeout=1s
```

These override everything except explicitly set system properties:

```
-Dapp.timeout=500ms
```

If both exist, `-D` wins over `--`.

---

# 6. Testing sources

Spring test annotations create temporary overrides.

### @TestPropertySource

```java
@TestPropertySource(properties = "app.timeout=50ms")
class MyTest {}
```

Overrides **everything** from YAML.

### @DynamicPropertySource  
Useful for dynamic values like container ports:

```java
@DynamicPropertySource
static void register(DynamicPropertyRegistry reg) {
  reg.add("db.url", () -> container.getJdbcUrl());
}
```

### @ActiveProfiles  
Activates profile files inside `src/test/resources`.

Example:

```
src/test/resources/application-test.yml
```

---

# 7. Multiple files under the same profile

Spring supports:

```
application.yml
application-dev.yml
config/application.yml
config/application-dev.yml
```

All files with the same name participate.  
Files in `/config/` override those in the root.

---

# 8. Externalized configuration (outside the JAR)

Useful for container/docker deployments:

```
/etc/app/application.yml
/config/application-prod.yml
```

These override packaged YAML but still follow the same precedence rules.

Spring searches in order:

```
classpath:/  
classpath:/config/  
file:./  
file:./config/  
file:/config/
```

---

# 9. Diagnosing “where did this value come from?”

Enable configuration debugging:

```properties
logging.level.org.springframework.boot.context.config=DEBUG
logging.level.org.springframework.boot.context.properties=DEBUG
```

Then check actuator (if enabled):

```
/actuator/env
/actuator/configprops
```

`/actuator/env` tells you:
- value
- origin (file, env var, CLI arg)
- precedence

`/actuator/configprops` shows the resolved binding.

---

# 10. Layered override example (practical)

Given:

**application.yml**

```yaml
app.host: http://localhost
```

**application-dev.yml**

```yaml
app.host: http://dev.local
```

Env var:

```bash
APP_HOST=http://env.local
```

CLI:

```bash
--app.host=http://cli.local
```

System property:

```bash
-Dapp.host=http://sys.local
```

Final result:

```
sys.local  ← highest
```

Full order:

```
sys.local > cli.local > env.local > dev.local > application.yml
```

---

# 11. DO / DON'T / pitfalls

**DO**
- Keep `application.yml` minimal and generic.
- Put environment differences into `application-<profile>.yml`.
- Use environment variables for secrets in real deployments.
- Use command-line arguments for temporary override during development.

**DON'T**
- Don’t hardcode defaults in code when YAML could hold them.
- Don’t rely on the order of keys inside YAML (irrelevant).
- Don’t mix env vars and CLI args arbitrarily — use one strategy.

**Pitfalls**
- Wrong indentation in YAML causes silent missing keys.  
- Misnamed profile `application-prod.yml` vs `application-production.yml`.  
- Env var collisions: `APP_HOST` unintentionally overrides YAML.  
- Docker Compose `environment:` values *always* override.

---

# 12. Quick glossary

- **Config Data** — Spring Boot’s mechanism for loading YAML/properties in order.
- **Property Source** — A location containing key/value pairs (file, env var, CLI, etc.).
- **Precedence** — Which source wins when multiple define the same key.
- **Binding** — Mapping resolved values into Java types.

