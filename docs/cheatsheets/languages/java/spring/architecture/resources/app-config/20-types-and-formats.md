---
title: Types & Formats 
date: 2025-11-06
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for Spring Boot configuration property types and formats, detailing how YAML values map to Java types.
aliases:
  - Spring Properties - Types & Formats Cheatsheet
---


# Types & Formats — Cheatsheet

Spring Boot performs automatic type conversion when binding configuration values. YAML values are Strings, but the binder transforms them into Java types such as `Duration`, `DataSize`, `URI`, collections, and nested structures.

This page shows the formats Spring understands and how they map into Java types.

---

# 1. Primitives & simple types

## Strings
```yaml
app:
  title: "My App"
```

```java
String title;
```

## Integers & longs
```yaml
retries: 3
```

```java
int retries;
long retries;
```

## Booleans
```yaml
feature:
  signup: true
```

```java
boolean signup;
```

## Enums
```yaml
logging:
  level: "INFO"
```

```java
LogLevel level; // must match enum value
```

YAML values are case-insensitive for enums (`info` → `INFO`).

---

# 2. Duration

Supported units: `ns`, `us`, `ms`, `s`, `m`, `h`, `d`.

Examples:

```yaml
timeout: 5s
retryDelay: 250ms
session: 1h
```

Bound to:

```java
Duration timeout;
Duration retryDelay;
Duration session;
```

ISO-8601 also works:

```
PT5S     # 5 seconds
PT1H     # 1 hour
```

---

# 3. DataSize

Formats include `10KB`, `500MB`, `2GB`, case-insensitive.

```yaml
uploads:
  max: 20MB
  thumbnail: 200KB
```

Java:

```java
DataSize max;
DataSize thumbnail;
```

---

# 4. URI, URL, Path, InetAddress

```yaml
app:
  host: "https://api.example.com"
  cacheDir: "/var/tmp/cache"
  ip: "192.168.1.10"
```

Maps to:

```java
URI host;
Path cacheDir;
InetAddress ip;
```

Trailing slashes matter for `URI.resolve()`.  
Use a helper class (e.g., ProblemLinks) to normalize once.

---

# 5. Lists

Comma-separated:

```yaml
admins: alice,bob,charlie
```

Or array style:

```yaml
admins:
  - alice
  - bob
  - charlie
```

Java:

```java
List<String> admins;
List<URI> hosts;
```

---

# 6. Maps

```yaml
roles:
  admin: "alice"
  user: "bob"
```

```java
Map<String, String> roles;
```

Nested maps:

```yaml
limits:
  free:
    max: 10
  pro:
    max: 100
```

```java
Map<String, Limits> limits;
```

---

# 7. Nested objects / records

```yaml
app:
  host: "http://localhost:8080"
  limits:
    itemsPerPage: 25
    uploads: 10MB
```

```java
public record AppProps(URI host, Limits limits) {
  public record Limits(int itemsPerPage, DataSize uploads) {}
}
```

Spring detects nested structure from keys.

---

# 8. Collections of complex types

```yaml
servers:
  - host: https://a.example.com
    timeout: 5s
  - host: https://b.example.com
    timeout: 10s
```

```java
List<Server> servers;

public record Server(URI host, Duration timeout) {}
```

---

# 9. YAML anchors and reuse (optional but handy)

```yaml
defaults: &base
  timeout: 5s
  retries: 3

serviceA:
  <<: *base

serviceB:
  <<: *base
  retries: 5
```

Be cautious in large teams — anchors aren’t universally loved.

---

# 10. Environment variables → binding patterns

Spring maps env vars with this transformation:

```
MY_APP_HOST → app.host
MY_APP_LIMITS_UPLOADS → app.limits.uploads
```

Rules:
- underscores → dots  
- uppercase → lowercase  
- nested paths supported

Example:

```bash
export APP_LIMITS_ITEMS_PER_PAGE=10
```

YAML equivalent:

```yaml
app:
  limits:
    itemsPerPage: 10
```

---

# 11. Command-line arguments → binding

```bash
--app.host=http://override.local
--app.limits.itemsPerPage=5
```

These override YAML values.

---

# 12. Conversion pitfalls

- **Comma in lists**  
  `"a, b"` → list `["a", " b"]` (space preserved).  
  Prefer multi-line lists for clarity.

- **Trailing slash in URI**  
  `https://api.com` vs `https://api.com/` affects `resolve()` results.

- **Enum mismatch**  
  YAML must map to existing enum names or you get a startup failure.

- **Paths on Windows**  
  `C:\data\root` must be quoted inside YAML.

- **DataSize and Duration case sensitivity**  
  `10mb` works; `10Mb` works; `10mB` works. Mixed-case is fine.

---

# 13. Quick table (copy-paste reference)

| YAML value                     | Java type          | Example YAML         | Example Java                         |
|-------------------------------|---------------------|----------------------|---------------------------------------|
| `"hello"`                     | `String`            | `msg: "hello"`       | `String msg`                          |
| `5`                           | `int` / `long`      | `count: 5`           | `int count`                           |
| `true`                        | `boolean`           | `enabled: true`      | `boolean enabled`                     |
| `"INFO"`                      | `Enum`              | `level: INFO`        | `LogLevel level`                      |
| `10s`, `250ms`                | `Duration`          | `timeout: 10s`       | `Duration timeout`                    |
| `20MB`                        | `DataSize`          | `size: 20MB`         | `DataSize size`                       |
| `"https://api.com"`           | `URI`               | `host: https://a`    | `URI host`                            |
| `"/var/data"`                 | `Path`              | `root: /var/data`    | `Path root`                           |
| list (array)                  | `List<T>`           | `items: [1,2]`       | `List<Integer> items`                 |
| map                           | `Map<String,T>`     | `limits: {a:1,b:2}`  | `Map<String,Integer> limits`          |
| nested block                  | record/object       | `app: {host: ...}`   | `AppProps props`                      |

---

# 14. Troubleshooting conversion

Enable debug logs:

```properties
logging.level.org.springframework.boot.context.properties=DEBUG
logging.level.org.springframework.boot.context.config=DEBUG
```

Common failures:
- Wrong unit for `Duration` or `DataSize`
- Enum mismatch
- Wrong indentation in YAML
- Invalid URI (spaces, missing scheme)

---

# 15. Practical example (aligned with your AppProps)

**YAML**

```yaml
app:
  host: http://localhost:8080
  timeout: 5s
  admins:
    - alice@example.com
    - bob@example.com
  limits:
    uploads: 15MB
    itemsPerPage: 25
```

**Java**

```java
@ConfigurationProperties("app")
public record AppProps(
  URI host,
  Duration timeout,
  List<String> admins,
  Limits limits
) {
  public record Limits(DataSize uploads, int itemsPerPage) {}
}
```

Everything binds automatically.

