# 📚 Java Reflection & Class Loading Cheatsheet and Concept Blueprint

---

## 🧩 Suggested Folder Layout

```
docs/
└─ cheatsheets/
   ├─ languages/
   │  └─ java/
   │     ├─ core/
   │     │  ├─ reflection-basics.md
   │     │  ├─ classpath-and-classloader.md
   │     │  ├─ annotations.md
   │     │  ├─ jvm-runtime-environment.md
   │     │  └─ bytecode-and-compilation-flow.md
   │     ├─ spring/
   │     │  ├─ spring-reflection-and-autowired.md
   │     │  ├─ spring-bean-lifecycle.md
   │     │  ├─ spring-context-loading.md
   │     │  └─ spring-boot-run-sequence.md
   │     └─ advanced/
   │        ├─ custom-classloader-example.md
   │        └─ hot-reload-concept.md
   └─ concepts/
      ├─ java/
      │  ├─ what-is-reflection.md
      │  ├─ how-jvm-loads-classes.md
      │  ├─ runtime-vs-compile-time.md
      │  └─ jre-jvm-jdk-differences.md
      └─ spring/
         ├─ why-spring-uses-reflection.md
         ├─ dependency-injection-mechanics.md
         └─ how-annotations-drive-spring.md
```

---

## 🧭 Content Breakdown and Key Ideas

### 1. **Reflection Basics (cheatsheet)**

Focus: Practical syntax and examples.

Sections:

* What reflection is.
* Core classes (`Class`, `Field`, `Method`, `Constructor`).
* Reading annotations.
* Creating instances with reflection.
* When to use / not to use.

---

### 2. **What Is Reflection (concept)**

Focus: Explanation in plain language.

* Idea: Java program inspecting itself.
* Why frameworks need it.
* Example of simple use (`Class.forName`, `.newInstance()`).
* Advantages and trade-offs.

---

### 3. **Classpath & ClassLoader (cheatsheet)**

Focus: Practical reference.

* What “classpath” means.
* Compile vs runtime classpath.
* `java -cp`, Maven/Gradle defaults.
* Layers of class loaders (bootstrap, platform, app).
* Typical errors (`ClassNotFoundException`, `NoClassDefFoundError`).

---

### 4. **How JVM Loads Classes (concept)**

Focus: Mechanism & lifecycle.

* Step-by-step from command to `main`.
* Class loading, linking, initialization.
* `ClassLoader` hierarchy.
* When reflection happens during loading.

---

### 5. **JVM Runtime Environment (cheatsheet)**

* JVM vs JRE vs JDK.
* Runtime memory areas (heap, method area, stack).
* Execution process (load → verify → execute).

---

### 6. **Spring Boot Run Sequence (cheatsheet)**

Show the journey of `SpringApplication.run()`:

* Bootstrapping context.
* Component scan.
* Bean creation.
* Dependency injection.
* Ready state.

---

### 7. **Spring Reflection & @Autowired (cheatsheet)**

* How Spring reads annotations via reflection.
* How fields and constructors are injected.
* Order of resolution (`@Primary`, `@Qualifier`, type match).
* Simplified pseudo-code for reflection steps.

---

### 8. **Why Spring Uses Reflection (concept)**

* Dynamic discovery instead of hard-coding.
* Annotations as metadata.
* Inversion of Control (IoC).
* Trade-offs (speed, safety).

---

### 9. **Custom ClassLoader Example (advanced cheatsheet)**

* Minimal example code of a class loader.
* Step-by-step description of `defineClass`.
* Use case in plugin systems or frameworks.

---

### 10. **Hot Reload Concept (advanced concept)**

* How reloading works (new class loader, drop old references).
* Relation to Spring Boot DevTools and Tomcat.
* Memory & isolation considerations.

---

### 11. **Runtime vs Compile-time (concept)**

* Difference in who “knows” what when.
* Why reflection and annotations belong to runtime.
* How compile-time tools (e.g., Lombok) differ.

---

### 12. **Bean Lifecycle (cheatsheet)**

* Creation → dependency injection → initialization → destruction.
* Hooks (`@PostConstruct`, `@PreDestroy`).
* Where reflection enters each phase.

---

### 13. **Dependency Injection Mechanics (concept)**

* How Spring resolves dependencies.
* Bean scopes, qualifiers, primaries.
* Circular dependencies and lazy initialization.

---

### 14. **How Annotations Drive Spring (concept)**

* Annotations as configuration language.
* Scanning and metadata parsing.
* Comparison to XML config.

---

### 15. **JRE, JVM, JDK Differences (concept)**

* Short overview:
  JDK = development kit,
  JRE = runtime environment,
  JVM = execution engine.
* Which one includes which.

---

## 🧠 How to Write Them

* **Cheatsheets**: fast lookup; short examples, commands, diagrams.
* **Concepts**: calm explanations; analogies and short code snippets.

Keep each file self-contained — someone reading it six months later should not need context.

---



docs(algomonster): add problem "parking spot" descriptions for "OOP Design" section.
feat(playing-cards-java): add practice implementation with solution and tests.
