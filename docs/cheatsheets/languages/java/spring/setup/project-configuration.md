---
title: Project Configuration
date: 2025-10-22
tags: 
    - spring
    - java
    - configuration
summary: Best practices for managing file paths and directories in Spring Boot applications using externalized configuration and runtime Path resolution.
aliases:
    -  Project Configuration
---

# 🧭 Project Configuration & Path Management — The Complete Cheatsheet

> **Goal:** Manage configuration in a way that is **portable**, **safe**, and **predictable** across environments — from local dev to containers and cloud.

---

## 1. Core Principles

1. **Externalize configuration** — keep code constant; change values via YAML, profiles, or environment variables.
2. **Single source of truth** — for every property, define exactly one authoritative source at runtime.
3. **Typed + validated** — define schema and validate at startup.
4. **Profile-based environments** — dev, test, staging, prod.
5. **Never hard-code paths** — resolve `Path`s dynamically at startup.
6. **Immutable builds, mutable config** — identical JARs, different configs.
7. **Expose effective config** — redacted, layered view for observability.

---

## 2. Configuration Sources & Precedence

**Typical precedence (low → high):**
`defaults.yml` → `application-{profile}.yml` → `local.yml` → env vars → CLI/system props → secrets manager.

**Locations:**

* Defaults: `src/main/resources/application.yml`
* Profiles: `application-prod.yml`
* Local overrides: `.env` or `application-local.yml`
* Secrets: `/run/secrets/*` or cloud secret store.

---

## 3. Spring Boot Binding — The Foundation

Spring Boot automatically maps config values to fields.

### `@Value`

For one-off keys:

```java
@Value("${paths.entries.uploads:data/uploads}")
private String uploadsDir;
```

### `@ConfigurationProperties`

For structured configs:

```java
@ConfigurationProperties(prefix = "paths")
public record PathsProps(String home, Map<String,String> entries) {}
```

Environment variable mapping: `PATHS_ENTRIES_UPLOADS → paths.entries.uploads`.

---

## 4. File Storage Path Management (Practical Example)

Define in `application.yml`:

```yaml
myapp:
  home: "${user.home}/IdeaProjects/playground/contacts"
  photos-dir: "data/photos"
```

**Properties class:**

```java
@Component
@ConfigurationProperties(prefix = "myapp")
public class MyAppProperties {
  private String home;
  private String photosDir;

  public Path appHomePath() {
    return Path.of(home).toAbsolutePath().normalize();
  }
  public Path photosPath() {
    Path p = Path.of(photosDir);
    return p.isAbsolute() ? p.toAbsolutePath().normalize()
                          : appHomePath().resolve(p).toAbsolutePath().normalize();
  }
}
```

**File storage service:**

```java
@Service
public class FileStorageService {
  private final Path photos;

  public FileStorageService(MyAppProperties props) throws IOException {
    this.photos = props.photosPath();
    Files.createDirectories(this.photos);
  }

  public Path savePhoto(String filename, byte[] bytes) throws IOException {
    Path target = photos.resolve(filename).normalize();
    if (!target.startsWith(photos)) throw new IllegalArgumentException("Invalid filename");
    return Files.write(target, bytes);
  }
}
```

**Best practices:**

* Validate directory existence on startup.
* Log absolute resolved paths.
* Allow environment overrides (`MYAPP_HOME`, `MYAPP_PHOTOS_DIR`).

---

## 5. Generalized Path Registry Pattern

One central class to manage *all* logical paths — uploads, logs, cache, configs, etc.

**application.yml:**

```yaml
paths:
  home: "${user.home}/IdeaProjects/playground/myapp"
  entries:
    uploads: "data/uploads"
    cache: "var/cache"
    logs: "/var/log/myapp"
    keystore: "config/keystore.p12"
```

**Properties:**

```java
@Component
@ConfigurationProperties(prefix = "paths")
public class PathsProperties {
  private String home;
  private Map<String, String> entries;

  public Path homePath() { return Path.of(home).toAbsolutePath().normalize(); }
  public Map<String, String> entries() { return Map.copyOf(entries); }
}
```

**Registry:**

```java
@Component
public class PathRegistry {
  private final Path home;
  private final Map<String, String> raw;

  public PathRegistry(PathsProperties props) {
    this.home = props.homePath();
    this.raw = props.entries();
  }

  public Path resolve(String key) {
    String v = raw.get(key);
    if (v == null) throw new IllegalArgumentException("Unknown path: " + key);
    Path p = Path.of(v);
    return (p.isAbsolute() ? p : home.resolve(p)).toAbsolutePath().normalize();
  }

  public Path ensureDir(String key) throws IOException {
    Path dir = resolve(key);
    Files.createDirectories(dir);
    return dir;
  }

  public Path safeChild(String dirKey, String filename) {
    Path base = resolve(dirKey);
    Path child = base.resolve(filename).normalize();
    if (!child.startsWith(base)) throw new IllegalArgumentException("Traversal: " + filename);
    return child;
  }
}
```

**Usage:**

```java
@Service
public class CacheService {
  private final Path cache;
  public CacheService(PathRegistry paths) throws IOException {
    this.cache = paths.ensureDir("cache");
  }
}
```

---

## 6. Validation & Observability

### CommandLineRunner — log + validate at startup

```java
@Component
@Order(10)
public class PathsStartupRunner implements CommandLineRunner {
  private static final Logger log = LoggerFactory.getLogger(PathsStartupRunner.class);
  private final PathRegistry paths;
  private final List<String> requiredDirs = List.of("uploads", "cache", "temp");

  public PathsStartupRunner(PathRegistry paths) { this.paths = paths; }

  @Override
  public void run(String... args) throws Exception {
    log.info("=== Resolving paths ===");
    for (String key : requiredDirs) {
      Path dir = paths.ensureDir(key);
      log.info("paths.{} -> {}", key, dir);
      if (!Files.isWritable(dir))
        throw new IllegalStateException("Not writable: " + dir);
    }
    log.info("=== Path validation complete ===");
  }
}
```

### HealthIndicator — continuous liveness/readiness

```java
@Component("paths")
public class PathsHealthIndicator implements HealthIndicator {
  private final PathRegistry paths;
  private final String[] keys = {"uploads", "cache", "temp"};

  public PathsHealthIndicator(PathRegistry paths) { this.paths = paths; }

  @Override
  public Health health() {
    Map<String, Object> details = new LinkedHashMap<>();
    boolean allOk = true;
    for (String key : keys) {
      Path p = paths.resolve(key);
      boolean exists = Files.exists(p);
      boolean dir = exists && Files.isDirectory(p);
      boolean writable = dir && Files.isWritable(p);
      details.put(key, Map.of("path", p.toString(), "exists", exists, "dir", dir, "writable", writable));
      allOk &= exists && dir && writable;
    }
    return allOk ? Health.up().withDetails(details).build()
                 : Health.down().withDetails(details).build();
  }
}
```

Enable Actuator:

```yaml
management:
  endpoints.web.exposure.include: health,info
  endpoint.health.show-details: when_authorized
```

---

## 7. Path & Directory Rules (Cross‑Platform)

* Always resolve to **absolute** paths at startup.
* Normalize (`..`, `.`), unify separators, avoid trailing slashes.
* Use **API joins** (`Path.resolve`, not string concatenation).
* Avoid non‑ASCII or space characters for portability.
* Containers: mount writable volumes; never write inside image filesystem.
* Respect XDG dirs (`~/.config`, `~/.cache`) for user apps.

**Environment vars mapping:**

```bash
PATHS_HOME=/data
PATHS_ENTRIES_UPLOADS=data/uploads
PATHS_ENTRIES_LOGS=/var/log/myapp
```

**Dockerfile:**

```dockerfile
ENV PATHS_HOME=/data
ENV PATHS_ENTRIES_UPLOADS=data/uploads
ENV PATHS_ENTRIES_CACHE=var/cache
```

---

## 8. Quick Spring Boot Notes

* File order: `application.yml` < profile YAML < env < system props < CLI.
* Use `${VAR:default}` placeholders.
* `spring.config.import=optional:file:…` to add layered configs.
* `@ConfigurationProperties` = typed binding; use `@Validated` for constraints.
* Default to stdout logging in containers; configurable via `logging.file.name`.

---

## ✅ Summary

* Keep **logical** paths in config, not code.
* Bind them via `@ConfigurationProperties`.
* Resolve into **absolute, normalized** `Path`s.
* Validate + log at startup (fail fast).
* Use HealthIndicator for runtime monitoring.
* Containers + env vars should map cleanly.
* Result: portable, observable, safe configuration across every environment.
