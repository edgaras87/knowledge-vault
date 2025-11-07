---
title: Resources vs Config
date: 2025-11-03
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: Why put config classes in `config/` package vs `resources/` folder?
aliases:
  - Resources vs Config Cheatsheet
---



# Why `resources/` vs `config/` package for config classes?

Two different “config worlds” here, and they seem to be bumping shoulders in your head.

`resources/` is **configuration data** (YAML / properties).

`config/` in Java packages is **configuration code** (your typed bindings, beans).

Think of it like a brain and its memories:

* `src/main/resources/` = memory files (“what the world *is* right now”)
* `src/main/java/.../config/` = the cognitive machinery reading them (“how to *interpret* those memories”)

If they lived in one folder, the meaning gets muddy. It's like putting receipts, cooking recipes, and your neural circuits in the same drawer. Sure, technically possible — but inconvenient for future-you.

---

### Why *not* put config class into `resources/`?

Because `resources/` is **not Java code territory**.
It’s pure data.
No classes should live there — just files Spring will read at runtime.

Spring doesn't scan `resources/` looking for beans or `@ConfigurationProperties`.

---

### Why `config/` package exists at all?

It's a **marker that says:**

> “This is where the application’s runtime settings live — in code form.”

It organizes:

* `@ConfigurationProperties` classes
* `@Bean` definitions
* maybe later a `CorsConfig`, `JacksonConfig`, `PasswordPolicyConfig`

It’s a *category*, not a rule mandated by Spring.

Could you put your config class in `shared/`, `infrastructure/`, or `application/`?
Sure — it’s all about clarity.

But `config` is a simple word that future teammates (and future you with coffee shakes) understand instinctively.

---

### Quick analogy

*YAML = knobs on your stereo.*
*`config/` classes = the circuitry that reads knob positions and adjusts sound.*

You don't solder knobs inside the sticker drawer. Even if you could.

---

### Minimal mental model

**resources/config files = values**
**java/config code = structure + type safety**

They shake hands at startup and create your app’s environment.

That separation becomes more obvious when later you have:

```
src/main/resources/application-dev.yml
src/main/resources/application-prod.yml
```

and

```
src/main/java/.../config/SecurityConfig.java
src/main/java/.../config/ObjectMapperConfig.java
src/main/java/.../config/AppProps.java
```

Different layers, different roles.

---

### A philosophical bow-tie to tie it all up

Clean architecture always honors **separation of concerns**.
Spring pushes us gently in that direction.

Code config belongs with code.
Runtime values belong with runtime files.

Tiny distinction — huge clarity long-term.

