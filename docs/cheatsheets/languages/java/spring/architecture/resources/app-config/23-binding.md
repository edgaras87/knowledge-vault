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

# Property Binding — Cheatsheet

Binding is how Spring takes values from `application.yml`, environment variables, or command-line arguments and turns them into Java fields and records.  
Use it to provide structured, typed configuration to your application without hardcoding.

---

## 1. Two binding mechanisms

### a) @ConfigurationProperties  
Use this for **groups** of properties, nested values, structured config, and anything domain-like.

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

Or rely on `@SpringBootApplication` package scanning if the class is under the main package.

### b) @Value  
Use for **single**, simple values, especially when they have no structure.

```java
@Value("${app.host}")
URI host;
```

Don’t use `@Value` for multi-field config — it becomes brittle and noisy.

---

## 2. YAML → Java type matching

Spring converts common values automatically:

```yaml
app:
  host: "http://localhost:8080"
  timeout: 5s
  limits:
    itemsPerPage: 20
```

Maps into:

```java
new AppProps(
  URI("http://localhost:8080"),
  Duration.ofSeconds(5),
  new Limits(20)
);
```

Supported types include:
- `String`, `int`, `long`, `boolean`
- `Duration`, `Period`, `DataSize`
- `URI`, `URL`, `InetAddress`
- `List<T>`, `Set<T>`, `Map<String, T>`
- Records and simple POJOs

---

## 3. Constructor binding (recommended)

Records use constructor binding automatically.

For classes:

```java
@ConfigurationProperties(prefix = "app")
@ConstructorBinding
public class AppProps {
  private final URI host;
  public AppProps(URI host) { this.host = host; }
}
```

Constructor binding = immutable + safe + clear.

---

## 4. Nested structures

Just define nested records or static inner classes:

```java
@ConfigurationProperties("storage")
public record StorageProps(Path root, S3 s3) {
  public record S3(String bucket, String region) {}
}
```

YAML:

```yaml
storage:
  root: "/var/data"
  s3:
    bucket: "media"
    region: "eu-central-1"
```

---

## 5. Lists and maps

**List**

```yaml
app:
  admins:
    - alice
    - bob
```

```java
List<String> admins
```

**Map**

```yaml
limits:
  user: 10
  admin: 100
```

```java
Map<String, Integer> limits
```

---

## 6. Default values

Option A — define defaults in Java:

```java
public record AppProps(URI host, Duration timeout) {
  public AppProps {
    if (timeout == null) timeout = Duration.ofSeconds(5);
  }
}
```

Option B — define defaults in YAML (preferred):

```yaml
app:
  timeout: 5s
```

YAML defaults keep Java clean.

---

## 7. Overriding & precedence (binding sees final resolved value)

Binding doesn’t care *where* the value came from — it receives the final resolved property.

Precedence example:

1) Command line  
2) Environment variables  
3) application-<profile>.yml  
4) application.yml  

So this changes the final `AppProps.host` regardless of YAML:

```bash
java -jar app.jar --app.host=https://override.example
```

---

## 8. Binding errors & diagnostics

If Spring cannot convert a value, it fails fast:

```
Failed to bind properties under 'app.timeout' to java.time.Duration
```

To debug:

```properties
logging.level.org.springframework.boot.context.properties=DEBUG
logging.level.org.springframework.boot.context.config=DEBUG
```

---

## 9. Mixing @Value and @ConfigurationProperties

Allowed, but separate concerns cleanly.

Example of sensible mixing:

- Structured config → `AppProps`  
- Single flag → `@Value("${feature.preview:false}")`

Avoid mixing them for the same domain.

---

## 10. Testing property binding

```java
@SpringBootTest
@EnableConfigurationProperties(AppProps.class)
@TestPropertySource(properties = "app.timeout=1s")
class AppPropsTest {

  @Autowired AppProps props;

  @Test void bindsCorrectly() {
    assertEquals(Duration.ofSeconds(1), props.timeout());
  }
}
```

---

## 11. Practical example (in your style)

YAML:

```yaml
app:
  host: http://localhost:8080
  limits:
    uploads: 10MB
    itemsPerPage: 25
```

Java:

```java
@ConfigurationProperties(prefix = "app")
public record AppProps(URI host, Limits limits) {
  public record Limits(DataSize uploads, int itemsPerPage) {}
}
```

Usage:

```java
@RestController
class AppController {
  private final AppProps props;

  AppController(AppProps props) {
    this.props = props;
  }

  @GetMapping("/api/app")
  AppInfoResponse info() {
    return new AppInfoResponse(props.host(), props.limits().itemsPerPage());
  }
}
```

---

## 12. DO / DON'T

**DO**
- Use records for immutable config.
- Keep each configuration class small and grouped by domain.
- Use YAML over properties when nesting is involved.

**DON’T**
- Don’t scatter `@Value` everywhere.
- Don’t put secrets into YAML.
- Don’t use binding for runtime state (properties are startup config).

---

## 13. Quick glossary

- **Binder** — converts raw String values into Java types.  
- **ConfigurationProperties** — structured binding of grouped values.  
- **@Value** — single-value injection.  
- **Config Data** — system that loads YAML/env/args in a defined order.

