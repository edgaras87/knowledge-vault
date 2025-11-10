---
title: Content
date: 2025-11-07
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: An index of cheat sheets covering various aspects of Spring Boot's configuration properties, including profiles, binding, validation, types, sources, exposure, feature flags, secrets, testing, and troubleshooting.
aliases:
  - Spring Properties - Content Cheatsheet
---

# Spring Properties — Index

Spring Boot’s configuration system handles values from YAML, environment variables, profiles, and other sources.  
This section groups everything into focused notes, each covering a specific theme: profiles, binding, validation, types, sources, exposure, feature flags, secrets, testing, and troubleshooting.

Use this index as the entry point when navigating or learning any part of Spring’s configuration model.



---

# 1. Types & Formats

Reference for how Spring converts text into Java types.

In **types-and-formats.md** covers:

- Duration (`10s`, `2m`, `500ms`)
- DataSize (`10MB`, `200KB`)
- URI, InetAddress, Path
- lists and maps
- structured objects
- environment variable mapping
- quick reference table

---

# 2. Binding  

How values move from YAML/env → Java records/classes.

In **binding.md** covers:

- `@ConfigurationProperties`
- `@Value`
- constructor binding
- nested records
- list/map binding
- default values

---

# 3. Validation

Startup-time guarantees for configuration correctness.

In **validation.md** covers:

- `@Validated`
- `@NotNull`, `@Min`, etc.
- nested validation via `@Valid`
- fail-fast behavior
- validating structured config

---

# 4. Profiles  

How Spring selects environment-specific configuration.

In **profiles.md** covers:

- `spring.profiles.active`
- `application-<profile>.yml`
- profile groups
- profile-based activation of beans
- profile behavior in tests


---

# 5. Sources & Precedence

Understanding where values come from and which one wins.

In **sources-and-precedence.md** covers:

- complete precedence ladder
- YAML merge rules
- profiles + config data loader
- env var → property name mapping
- CLI + system property overrides
- `/actuator/env` debugging

---

# 6. Exposing Configuration

How to publish selected config to the outside world safely.

In **exposing-config.md** covers:

- shaping API DTO responses
- avoiding leaks of sensitive/internal values
- using `ProblemLinks` for RFC-7807 types
- controller patterns for `/api/app`
- normalizing URIs

---

# 7. Feature Flags  

Lightweight toggles to control behavior.

In **feature-flags.md** covers:

- boolean flags in YAML
- structured feature groups (`app.feature.*`)
- `@ConditionalOnProperty`
- profile-based vs flag-based decisions
- multi-state flags (enums)

---

# 8. Secrets  

Safe handling of passwords, API keys, and sensitive values.

In **secrets.md** covers:

- environment variables
- externalized secrets
- Compose/Kubernetes patterns
- masking logs
- forbidden locations (`application.yml`)

---

# 9. Testing

How to test binding, overrides, profiles, and dynamic configs.

In **testing.md** covers:

- `@TestPropertySource`
- `@ActiveProfiles`
- `DynamicPropertySource`
- ApplicationContextRunner
- testing conditional beans
- testing config exposure endpoints

---

# 10. Troubleshooting  

Diagnosis of binding issues, overrides, and config inconsistencies.

In **troubleshooting.md** covers:

- enabling config debug logs
- binding error analysis
- type mismatch debugging
- profile mixups
- environment-variable conflicts
- YAML indentation issues
- override collisions

---

# Suggested structure in the vault

```
spring/
└─ properties/
   ├─ index.md
   ├─ types-and-formats.md
   ├─ binding.md
   ├─ validation.md
   ├─ profiles.md
   ├─ sources-and-precedence.md
   ├─ exposing-config.md
   ├─ feature-flags.md
   ├─ secrets.md
   ├─ testing.md
   └─ troubleshooting.md
```

Everything here is designed to stand alone but also form a complete picture of Spring Boot’s configuration system.

