---
title: Property Binding 
date: 2025-11-06
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for Spring Boot property binding, detailing how to bind configuration properties from YAML to Java classes using @ConfigurationProperties and @Value.
aliases:
  - Spring Properties - Property Binding Cheatsheet
---

# Spring Boot Configuration Binding — Cheatsheet

Configuration binding loads values from YAML, environment variables, or command-line arguments and turns them into typed Java objects. Use it to centralize configuration, enforce structure, and keep runtime settings out of code.

---

## 1. Binding mechanisms

### `@ConfigurationProperties`

For grouped, structured, nested configuration.

```java
@ConfigurationProperties(prefix = "app")
public record AppProps(URI host, Duration timeout, Limits limits) {
  public record Limits(int itemsPerPage) {}
}
```

Enable via:

```java
@Configuration
@EnableConfigurationProperties(AppProps.class)
class Config {}
```

Or via `@ConfigurationPropertiesScan` inside the application package.

### `@Value`

For single, isolated values only.

```java
@Value("${feature.preview:false}")
boolean preview;   // includes default
```

Avoid using `@Value` for anything structured or multi-field.

---

## 2. YAML → Java conversion

```yaml
app:
  host: "http://localhost:8080"
  timeout: 5s
  limits:
    itemsPerPage: 20
```

Maps to:

```java
new AppProps(
  URI.create("http://localhost:8080"),
  Duration.ofSeconds(5),
  new Limits(20)
);
```

The binder supports primitives, wrappers, `URI`, `Duration`, `Period`, `DataSize`, `UUID`, collections, and simple POJOs/records.

---

## 3. Constructor binding

Records bind via their canonical constructor automatically.

```java
@ConfigurationProperties("app")
public record AppProps(URI host) {}
```

For classes:

```java
@ConfigurationProperties("app")
@ConstructorBinding
public class AppProps {
  private final URI host;
  public AppProps(URI host) { this.host = host; }
}
```

---

## 4. Nested configuration

Use nested records or static nested classes.

```java
@ConfigurationProperties("storage")
public record StorageProps(Path root, S3 s3) {
  public record S3(String bucket, String region) {}
}
```

---

## 5. Collections

**List**

```yaml
admins:
  - alice
  - bob
```

```java
List<String> admins;
```

**Map**

```yaml
limits:
  user: 10
  admin: 100
```

```java
Map<String, Integer> limits;
```

---

## 6. Defaults

### YAML-side defaults

Recommended for clarity.

```yaml
feature:
  preview: false
```

### Java-side defaults in configuration properties

```java
public record FeatureProps(
  @DefaultValue("false") boolean preview
) {}
```

### One-off scalar via `@Value`

```java
@Value("${feature.preview:false}")
boolean preview;
```

---

## 7. Property source precedence

1. Command line
2. Environment variables
3. `application-<profile>.yml`
4. `application.yml`

---

## 8. Binding errors & debugging

```
Failed to bind properties under 'app.timeout' to java.time.Duration
```

Diagnostics:

```
logging.level.org.springframework.boot.context.properties=DEBUG
logging.level.org.springframework.boot.context.config=DEBUG
```

---

## 9. Annotations for configuration binding

### Core

* `@ConfigurationProperties(prefix = "...")`
* `@ConfigurationPropertiesScan`
* `@EnableConfigurationProperties`
* `@Validated`

### Constructor control

* `@ConstructorBinding` (classes only)

### Key & default customization

* `@Name("external-key")`
* `@DefaultValue("...")`

### Units

* `@DurationUnit(...)`
* `@DataSizeUnit(...)`

### Conversion & nesting

* `@NestedConfigurationProperty`
* `@ConfigurationPropertiesBinding`

### Single-value injection

* `@Value("${key:default}")`

---

## 10. Example — Full configuration using your YAML

### YAML

```yaml
app:

  host: "https://api.example.com"
  cache-dir: "/var/tmp/cache"
  ip: "192.168.1.10"

  api:
    version: v1

  pagination:
    default-size: 20
    max-size: 200

  features:
    signup: true
    metrics: true

  limits:
    uploads: 25MB
    items-per-page: 100
    timeout: 5s

  locale:
    default-tag: "en_US"

  default-roles: [USER, ADMIN]
  role-quotas:
    USER: 100
    ADMIN: 1000
  labels:
    env: "prod"
    region: "eu-central"
```

### Java

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProps(
    @NotNull URI host,
    @Name("cache-dir") Path cacheDir,
    @NotNull String ip,

    Api api,
    Pagination pagination,
    Features features,
    Limits limits,
    Locale locale,

    List<Role> defaultRoles,
    Map<Role, Integer> roleQuotas,
    Map<String, String> labels
) {

  public record Api(
      @NotBlank String version
  ) {}

  public record Pagination(
      @Name("default-size") @Min(1) int defaultSize,
      @Name("max-size") @Min(1) int maxSize
  ) {}

  public record Features(
      @DefaultValue("false") boolean signup,
      @DefaultValue("false") boolean metrics
  ) {}

  public record Limits(
      @DataSizeUnit(MEGABYTES) DataSize uploads,
      @Name("items-per-page") @Min(1) int itemsPerPage,
      @DurationUnit(SECONDS) Duration timeout
  ) {}

  public record Locale(
      @Name("default-tag") @NotBlank String defaultTag
  ) {}
}

enum Role { USER, ADMIN }
```

### Example of a one-off flag via `@Value`

```java
@Component
class Flags {
  @Value("${feature.preview:false}")
  boolean preview;
}
```

---

## 11. DO / DON’T

**DO**

* Use records for immutable config.
* Group config by domain.
* Prefer YAML defaults or `@DefaultValue`.
* Keep nested config modeled as nested records.

**DON’T**

* Don’t scatter `@Value` everywhere.
* Don’t store secrets in YAML.
* Don’t mutate configuration objects after startup.

