---
title: Java — Future Blueprint
date: 2025-10-18
tags: 
    - cheatsheets
    - languages
    - java
summary: A scalable folder structure and evolution plan for organizing Java cheatsheets, accommodating core concepts, build tools, testing frameworks, and popular Java frameworks.
---



# 🧭 Java Cheatsheet Blueprint — Folder Layout & Evolution Plan

**Purpose:**  
This document defines the long-term structure for organizing Java-related cheatsheets.  
It starts simple but scales gracefully as new frameworks and tools appear.

---

## 📁 Planned Folder Layout

```

docs/
└─ cheatsheets/
└─ languages/
└─ java/
├─ index.md
├─ setup/
│  ├─ jdk-install.md        # SDKMAN!, PATH, versions
│  └─ project-structure.md  # src/main/java, resources, etc.
├─ core/
│  ├─ syntax.md
│  ├─ collections.md
│  ├─ streams.md
│  ├─ concurrency.md
│  ├─ records.md
│  └─ io-nio.md
├─ build/
│  ├─ maven/
│  │  ├─ basics.md
│  │  ├─ dependencies.md
│  │  ├─ plugins.md
│  │  └─ profiles.md
│  └─ gradle/
│     ├─ basics.md
│     ├─ dependencies.md
│     ├─ tasks.md
│     └─ multi-project.md
├─ testing/
│  ├─ junit5.md
│  ├─ mockito.md
│  └─ testcontainers.md
├─ frameworks/
│  ├─ spring/
│  │  ├─ boot/
│  │  │  ├─ starters.md
│  │  │  ├─ config.md
│  │  │  └─ actuator.md
│  │  ├─ web/
│  │  │  ├─ rest-controller.md
│  │  │  └─ validation.md
│  │  ├─ data/
│  │  │  ├─ jpa.md
│  │  │  └─ transactions.md
│  │  ├─ security/
│  │  │  ├─ basics.md
│  │  │  └─ jwt.md
│  │  └─ testing.md
│  ├─ micronaut/
│  └─ quarkus/
└─ tooling/
├─ jdk-tools.md
└─ lombok.md

```

---

## 🧠 Design Principles

### 1. Clear Separation of Concerns
- **Core Java** concepts live in `core/` — language fundamentals.
- **Frameworks** (Spring, Micronaut, Quarkus) live under `frameworks/`. Each gets its own namespace.
- **Build tools** (Maven, Gradle) belong to `build/`, since they’re language-tied.
- **Testing tools** (JUnit, Mockito, Testcontainers) share a top-level folder — they cross all projects.

### 2. Scalable and Future-Proof
- Spring starts as the main resident of `frameworks/`.
- Micronaut and Quarkus get empty placeholders now — ready for future growth.
- If Spring grows large, subdivide it (`boot/`, `web/`, `data/`, `security/`, `testing/`).

### 3. Flexibility for JVM Ecosystem
- If you later add Kotlin or Scala, they’ll live beside Java under `languages/`.
- Common JVM topics (like Gradle multi-language builds) can move to  
  `docs/concepts/build-systems.md` for shared knowledge.

---

## 🚀 Evolution Notes

- Keep the **first version simple** — start with `core/`, `setup/`, and `frameworks/spring/boot/`.
- As you expand, let **structure follow growth**, not theory.
- When duplication or overlap emerges between frameworks, extract those notes into:

```
docs/concepts/frameworks/java-frameworks-comparison.md
```


---

## 💡 Philosophy

A good cheatsheet library grows *organically*, not architecturally.  
Structure should reveal what you’ve learned — not cage what you haven’t yet.  
Build clarity first; hierarchy will follow naturally.


