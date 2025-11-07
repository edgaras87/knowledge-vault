---
title: Q&A's 
date: 2025-11-06
tags: 
    - spring
    - architecture
    - resources
    - cheatsheet
summary: A cheat sheet answering common questions about what belongs in the `src/main/resources/` directory of a Spring application.
aliases:
  - What Belongs in src/main/resources/ Q&A Cheat Sheet

---

# `src/main/resources/` — Q&A Cheatsheet

A conversational map of what belongs in `resources/`, why it exists, and what you’ll actually use there in a Spring Boot project.

---

### **Q: What is `src/main/resources/` actually for?**

It’s the place for **runtime classpath assets**: files your application needs *while it runs*, not while it compiles.

Spring will package them into your JAR and make them accessible via:

* `classpath:...`
* automatic config loading
* `ResourceLoader`
* static file serving

Think of it as your app’s “backpack” — only stable, lightweight items should go in.

---

### **Q: Is YAML the main inhabitant inside resources?**

YAML is common because Spring looks for:

* `application.yml`
* `application-*.yml`

But resources is **not** a YAML folder. It’s a runtime asset folder.
YAML just happens to be the main configuration format.

---

### **Q: What other config files can live here?**

Several kinds:

* `application.yml` / `.properties`
* `logback-spring.xml` or `log4j2.xml`
* `banner.txt`
* `messages.properties` (i18n)
* Custom `.json` or `.yml` if you load them manually

Resources stores configuration files that the runtime must read.

---

### **Q: Where do Flyway migrations live?**

Right here:

```
src/main/resources/db/migration/
```

Spring Boot + Flyway will automatically scan it and apply:

```
V1__init.sql
V2__add_categories.sql
V3__add_users.sql
```

This is one of the most important reasons the `resources/` folder exists.

---

### **Q: Should I store schema.sql or data.sql here?**

Only if you want Spring Boot’s auto-initializer (which runs before Flyway) — but in real backend apps, Flyway is the preferred approach.

If you use schema.sql/data.sql, they go at root:

```
src/main/resources/schema.sql
src/main/resources/data.sql
```

But avoid using both Flyway and schema.sql unless you know exactly why.

---

### **Q: Can I put static HTML or images here?**

Yes, under:

```
src/main/resources/static/
```

Spring serves files under `/static/**` directly over HTTP.

Typical uses:

* `/docs/**`
* `/problems/**` (for ProblemDetail type URIs)
* `/index.html`
* Logos, favicons, assets

Even backend-only APIs sometimes ship small static problem documentation.

---

### **Q: What about template engines (Thymeleaf, Freemarker)?**

Templates go here:

```
src/main/resources/templates/
```

Spring auto-loads them.
But if you're building pure REST APIs, this folder stays empty.

---

### **Q: Where do i18n message bundles go?**

Right here:

```
src/main/resources/messages.properties
src/main/resources/messages_en.properties
src/main/resources/messages_de.properties
```

Spring loads these automatically for:

* validation messages
* error messages
* UI/Backoffice text if you ever need it

Bean Validation often relies on this folder.

---

### **Q: What is `META-INF` and when would I touch it?**

`META-INF/` contains metadata that Spring or the JVM reads.

Examples:

```
src/main/resources/META-INF/spring.factories
src/main/resources/META-INF/additional-spring-configuration-metadata.json
src/main/resources/META-INF/spring-devtools.properties
```

You touch it only if:

* you write your own Spring Boot starter
* you create reusable internal libraries
* you need custom metadata for IDE hints
* you modify devtools behavior

This is advanced territory, but it’s good to know it exists.

---

### **Q: Should secrets ever live in resources?**

Never.

Not passwords.
Not tokens.
Not connection strings containing credentials.

Use:

* env vars
* `.docker.env`
* Kubernetes Secrets
* `spring.config.import` with external files

Resources is for **non-sensitive** runtime artifacts.

---

### **Q: Can I put large data files here?**

No — your JAR will balloon.
Large binary datasets belong:

* on a CDN
* in object storage
* mounted via volume
* in an external config location

Resources is for small, text-friendly assets.

---

### **Q: Should I put JSON schemas or example payloads here?**

Yes, if they’re part of your runtime contract.

Common locations:

```
src/main/resources/schema/
src/main/resources/examples/
```

For example:

* Kafka event schemas (Avro, JSON Schema)
* Example request/response bodies for docs
* OpenAPI fragments
* Contract test templates

These stay small and versioned with code.

---

### **Q: Can I load custom JSON/YAML files from resources at runtime?**

Yes:

```java
var resource = resourceLoader.getResource("classpath:example.json");
```

Or via:

```yaml
spring:
  config:
    import: classpath:/my-config.yml
```

But don’t abuse this — rely on `@ConfigurationProperties` first.

---

### **Q: Can resources contain per-profile overrides?**

Yes.

You can create:

```
application.yml
application-dev.yml
application-prod.yml
application-test.yml
```

Spring picks the correct file depending on the active profile.

---

### **Q: What’s the relationship between resources and the JAR file?**

Everything in `src/main/resources/` is copied into the final JAR under:

```
classes/
```

This is why:

* they must be small
* they must be stable
* they must be text-friendly

Resources are part of your **deployment artifact**.

---

### **Q: Is it good practice to generate files into resources during build?**

Usually no.

Generated code → inside `target/generated-sources/...`
Generated data → outside resources (unless tiny)
Generated docs → publish separately

Resources should remain static, predictable, and under version control.

---

### **Q: What about environment-specific secrets files?**

Put them *outside* the JAR and load them via:

```yaml
spring:
  config:
    import: optional:file:./secrets.yml
```

This keeps secrets out of version control and out of the JAR.

---

### **Q: When should I use classpath vs external file?**

**Classpath (`resources/`)**
Use it when the file must ship with the application build.

**External file / extra config location**
Use it when the file:

* changes per environment
* contains secrets
* is large
* is managed by ops, not devs

---

# Additional practical questions (common pain points)

### **Q: How do I see what properties Spring has actually loaded?**

Enable minimal Actuator:

```yaml
management.endpoints.web.exposure.include=env
```

Then:

```
GET /actuator/env
GET /actuator/env/app.host
GET /actuator/env/SPRING_DATASOURCE_URL
```

This is your “why isn’t my property taking effect?” detective tool.

---

### **Q: How do I override resources in tests?**

Use:

```
src/test/resources/
```

Spring’s test classpath shadows the main resources.

---

### **Q: Is there anything I should *not* put in resources even though it works?**

Yes:

* Java source files → should be in `src/main/java`
* Test data → should go in `src/test/resources`
* Build scripts → in project root
* Docker files → project root or `infra/`
* Temporary storage → never in classpath

Resources is not a general dump folder.

---

### **Q: What’s the easiest mental model?**

**Resources = the immutable runtime bundle that your app ships with.**

Only put things there that:

* must be packaged
* must be readable via classpath
* are small and text-like
* have stable meaning in production

Everything else goes somewhere else.

---
