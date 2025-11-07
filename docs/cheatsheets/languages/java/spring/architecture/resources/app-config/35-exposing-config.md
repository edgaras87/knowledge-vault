---
title: Exposing Config
date: 2025-11-06
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for safely exposing application configuration in Spring Boot applications, detailing best practices for creating public-facing DTOs and avoiding security pitfalls.
aliases:
    - Spring Properties - Exposing Config Cheatsheet
---

# Exposing Configuration — Cheatsheet

Sometimes an API needs to expose parts of the application’s configuration:  
version info, host URLs, limits, feature flags, or “app info” needed by clients.  
Expose **only what is meant to be public**, never raw `@ConfigurationProperties`,  
and avoid leaking secrets or internal paths.

This page shows safe, DTO-shaped configuration exposure using your layered structure.

---

# 1. Why not expose AppProps directly?

`AppProps` may contain:
- internal values (timeouts, internal hosts)
- sensitive values (tokens, secrets)
- irrelevant or confusing data for clients

Expose a **filtered DTO**, not the config class.

---

# 2. Safe pattern: controller → response DTO

Example minimal endpoint:

```java
@RestController
@RequestMapping("/api/app")
public class AppInfoController {
  private final AppProps props;

  public AppInfoController(AppProps props) {
    this.props = props;
  }

  @GetMapping
  public AppInfoResponse get() {
    return new AppInfoResponse(
      props.host(),
      props.limits().itemsPerPage()
    );
  }
}
```

DTO:

```java
public record AppInfoResponse(URI host, int itemsPerPage) {}
```

Only pick fields the outside world should see.

---

# 3. Keep DTOs in `web/<feature>/dto/response`

Structure:

```
src/main/java/com/example/app/web/appinfo/
├─ AppInfoController.java
└─ dto/
   └─ response/
      └─ AppInfoResponse.java
```

**Never** put DTOs in `config/`.  
`config/` contains *incoming* settings, not outward-facing shapes.

---

# 4. ProblemLinks: structured “problem type” URLs

When exposing error types (RFC-7807), don’t hardcode URLs everywhere.

Make a helper:

```java
@Component
public class ProblemLinks {
  private final URI base;

  public ProblemLinks(AppProps props) {
    // Normalize once
    URI raw = props.host();
    this.base = raw.toString().endsWith("/") ? raw : URI.create(raw + "/");
  }

  public URI base() { return base.resolve("problems/"); }

  public URI type(String slug) {
    return base().resolve(slug);
  }
}
```

Why this is good:
- Encapsulates URI normalization  
- Allows you to change the base in one place  
- Enables consistent ProblemDetail `type` URIs in error handlers  

Location:

```
src/main/java/.../web/common/ProblemLinks.java
```

or

```
src/main/java/.../config/ProblemLinks.java
```

Both are fine; keep it near config or the global exception handler.

---

# 5. Using ProblemLinks in exception handlers

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
  private final ProblemLinks links;

  public GlobalExceptionHandler(ProblemLinks links) {
    this.links = links;
  }

  @ExceptionHandler(DuplicateCategoryException.class)
  public ProblemDetail handle(DuplicateCategoryException ex, HttpServletRequest req) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setType(links.type("duplicate-category"));
    pd.setTitle("Duplicate category");
    pd.setDetail(ex.getMessage());
    pd.setInstance(URI.create(req.getRequestURI()));
    return pd;
  }
}
```

This ensures the API always produces stable, machine-readable URLs for problem types.

---

# 6. Avoid exposing full configprops

Spring Actuator offers:

```
/actuator/configprops  
/actuator/env
```

These are developer tools, not public API endpoints.  
They should be restricted or disabled in production.

If enabling Actuator for debugging:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: [env, configprops]
```

Protect behind authentication or IP restrictions in real deployments.

---

# 7. Shapes to use (DTOs)

### Singular configuration info  
When the client needs a compact app snapshot:

```java
public record AppInfoResponse(
  URI host,
  int itemsPerPage,
  boolean signupEnabled
) {}
```

### Nested shapes  
If multiple config groups must be exposed:

```java
public record AppInfoResponse(
  URI host,
  LimitsResponse limits,
  FeatureResponse feature
) {}

public record LimitsResponse(int itemsPerPage) {}
public record FeatureResponse(boolean signup) {}
```

### Never expose raw maps  
Always convert to shaped DTOs for stability.

---

# 8. DO / DON'T / pitfalls

**DO**
- Expose only public-facing config.
- Filter config into DTOs.
- Normalize URIs once in helper classes.
- Keep ProblemDetail `type` URLs stable and under a common base.

**DON'T**
- Don’t return `@ConfigurationProperties` directly.
- Don’t leak secrets, keys, internal endpoints, or credentials.
- Don’t compute ProblemDetail URIs inline in many places.

**Pitfalls**
- Forgetting trailing slash in base `URI` → broken `resolve()`.
- Returning full configprops in prod → security issue.
- Overexposing config meant for internal services only.

---

# 9. Practical example (your exact workflow)

Given:

```yaml
app:
  host: http://localhost:8080
  feature:
    signup: true
  limits:
    itemsPerPage: 25
```

Expose only:

```java
public record AppInfoResponse(
  URI host,
  boolean signupEnabled,
  int itemsPerPage
) {}
```

Controller:

```java
@GetMapping("/api/app")
public AppInfoResponse app() {
  return new AppInfoResponse(
    props.host(),
    props.feature().signup(),
    props.limits().itemsPerPage()
  );
}
```

This keeps the external shape stable even if you restructure AppProps internally.

---

# 10. Quick glossary

- **DTO** — Outbound shape for external clients.  
- **ProblemDetail** — Structured error response (RFC-7807).  
- **Type URI** — Unique identifier of an error category.  
- **Normalization** — Ensuring hosts and base paths are consistent.

