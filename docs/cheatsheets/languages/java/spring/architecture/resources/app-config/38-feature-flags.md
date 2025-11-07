---
title: Feature Flags  
date: 2025-11-06
tags: 
summary: 
aliases:
    - Spring Properties - Feature Flags Cheatsheet
---

# Feature Flags — Cheatsheet

Feature flags let you enable or disable functionality at runtime without code changes.  
Used correctly, they provide controlled rollout, A/B behavior, or environment-specific switches.  
In Spring Boot, flags are nothing more than **boolean configuration properties** plus optional conditional bean activation.

---

# 1. Simple boolean flags (recommended)

In `application.yml`:

```yaml
app:
  feature:
    signup: true
    metrics: false
```

Represent them in a typed config class:

```java
@ConfigurationProperties("app")
public record AppProps(Feature feature) {
  public record Feature(boolean signup, boolean metrics) {}
}
```

Usage in code:

```java
if (props.feature().signup()) {
  // allow signups
}
```

Why this is good:
- Visible in YAML  
- Testable  
- Works across environments  
- Easy for clients when exposing config

---

# 2. Conditional beans using @ConditionalOnProperty

Useful when a whole bean should exist only if a flag is “on”.

```java
@Configuration
@ConditionalOnProperty(
  prefix = "app.feature",
  name = "metrics",
  havingValue = "true",
  matchIfMissing = false
)
class MetricsConfig {
  @Bean
  MetricsCollector metricsCollector() { return new MetricsCollector(); }
}
```

This class loads only when:

```
app.feature.metrics=true
```

Good for:
- Enabling alternative service implementations  
- Registering debug tooling in dev  
- Loading optional integrations  

---

# 3. Flags via profiles (environment-level toggles)

When a feature should be enabled *only* in specific deployment environments:

**application-dev.yml**

```yaml
app:
  feature:
    debugEndpoints: true
```

**application-prod.yml**

```yaml
app:
  feature:
    debugEndpoints: false
```

Usage:

```java
if (props.feature().debugEndpoints()) { ... }
```

Profiles = “environment decides”.  
Booleans = “behavior decides”.  
Don’t mix them unless necessary.

---

# 4. Multi-value flags (string or enum switches)

Sometimes more than on/off:

```yaml
app:
  feature:
    mode: "experimental"
```

Java:

```java
public record Feature(Mode mode) {
  enum Mode { off, stable, experimental }
}
```

Usage:

```java
switch (props.feature().mode()) {
  case experimental -> ...
  case stable -> ...
}
```

This works well for:
- Choosing new algorithm internally  
- Gradual rollout states  
- Selecting between integrations

---

# 5. Flags in tests

Use lightweight overrides:

```java
@SpringBootTest
@TestPropertySource(properties = {
  "app.feature.signup=false"
})
class SignupFeatureTest {
  @Autowired AppProps props;

  @Test
  void signupDisabled() {
    assertFalse(props.feature().signup());
  }
}
```

Or activate a profile:

```
src/test/resources/application-test.yml
```

---

# 6. Exposing flags to clients

Expose selectively through DTOs:

```java
public record AppInfoResponse(boolean signupEnabled) {}
```

Avoid exposing internal or sensitive toggles such as:  
- debugging switches  
- integration toggles  
- experimental API previews  
- database migration modes

---

# 7. DO / DON'T / pitfalls

**DO**
- Use simple booleans for most toggles.  
- Keep all feature flags inside a single structured config group.  
- Treat flags as part of the “contract” of the app.  
- Use `@ConditionalOnProperty` for optional bean wiring.  
- Keep environment behavior predictable and consistent.

**DON'T**
- Don’t scatter flags across many YAML locations.  
- Don’t use flags instead of proper versioning.  
- Don’t put flags directly inside controllers.  
- Don’t make flags stateful — keep them immutable.  
- Don’t rely on profiles for small, behavior-only toggles.

**Pitfalls**
- Boolean negation confusion (`enabled=false` vs `disabled=true`).  
- Forgetting to add `matchIfMissing=false` → flag accidentally defaults to ON.  
- Overusing `@ConditionalOnProperty` → too many bean variants.  
- Inconsistent naming (`enableSignup`, `signupEnabled`, `signup`) — keep uniform.

---

# 8. Practical example (aligned with your workflow)

**application.yml**

```yaml
app:
  feature:
    signup: true
    experimentalMode: false
```

**Java**

```java
public record AppProps(Feature feature) {
  public record Feature(boolean signup, boolean experimentalMode) {}
}
```

**Usage (service or controller):**

```java
if (!props.feature().signup()) {
  throw new FeatureDisabledException("Signup disabled");
}
```

**Optional bean wiring:**

```java
@Configuration
@ConditionalOnProperty(prefix = "app.feature", name = "experimentalMode", havingValue = "true")
class ExperimentalConfig {
  @Bean ExperimentalService service() { return new ExperimentalService(); }
}
```

---

# 9. Quick glossary

- **Feature flag** — a configuration switch controlling behavior.  
- **Conditional bean** — a Spring bean enabled only when a condition is true.  
- **Toggle group** — structuring flags inside `app.feature.*`.  
- **Immutable flags** — flags read at startup, not dynamically changed.

