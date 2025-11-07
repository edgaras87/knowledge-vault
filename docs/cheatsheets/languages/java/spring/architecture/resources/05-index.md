---
title: Content
date: 2025-11-06
tags: 
    - spring
    - architecture
    - resources
    - cheatsheet
summary: A cheat sheet for what belongs in the `src/main/resources/` directory of a Spring application, detailing its purpose and common contents.
aliases:
    - Resource Content Cheatsheet
---

# Spring Architecture Cheat Sheet: `src/main/resources/`

The `src/main/resources/` directory is a strange little cave. YAML is only *one* of the creatures that live there. You’re standing at the mouth of the cavern now, and you’re asking a healthy question: *what else belongs in here? What deserves its own cheatsheet?*

Let’s pull back the curtains—casually but clearly—so you see what *belongs* in `resources/`, what *doesn’t*, and what parts of this cave you’ll likely fill as your project grows.

The tone stays friendly; the information stays sharp.

---

## `src/main/resources/` — Overview

This directory contains **runtime classpath resources**.
Everything placed here becomes part of the packaged application (JAR/WAR) and is accessible via the classpath at runtime.

`resources/` is for **static, non-secret, version-controlled assets** your application needs while running.

Typical contents:

```
src/main/resources/
├─ application.yml
├─ application-<profile>.yml
├─ db/
│  └─ migration/
├─ static/
├─ templates/
├─ messages.properties
└─ META-INF/
```

---

## Purpose of `resources/`

* Provide application configuration (`application.yml`, profile files).
* Provide migration files for DB schema tools (Flyway, Liquibase).
* Provide logging configuration (e.g., `logback-spring.xml`).
* Provide static assets to be served by Spring Boot.
* Provide message bundles for internationalization.
* Provide framework metadata (`META-INF/`).
* Provide small, text-based auxiliary files needed at runtime.

Everything here is included in the build artifact and available under `classpath:`.

---

## What belongs here

### **1. Application configuration**

Files automatically loaded by Spring Boot:

* `application.yml`
* `application-<profile>.yml` (dev, prod, test, staging, etc.)
* `.properties` equivalents
* Custom config imported via `spring.config.import`

These define environment-specific behavior.

---

### **2. Database migrations**

Location for Flyway migrations:

```
src/main/resources/db/migration/
    V1__init.sql
    V2__add_tables.sql
```

Spring Boot will run these automatically on startup.

---

### **3. Logging configuration**

Logging config loaded automatically from the classpath:

* `logback-spring.xml`
* `logback.xml`
* `log4j2.xml`

Controls log formatting, levels, appenders, and per-environment overrides.

---

### **4. Static assets**

Spring automatically serves resources from:

```
src/main/resources/static/
```

Common uses:

* API documentation files (`/docs/**`)
* ProblemDetail documentation (`/problems/**`)
* Images, CSS, JS (if needed)
* Health/status landing pages

These files are publicly accessible via HTTP.

---

### **5. Templates** (if used)

For template engines:

```
src/main/resources/templates/
```

Used by Thymeleaf, Freemarker, Mustache, etc.
REST APIs typically leave this empty.

---

### **6. Internationalization (i18n) bundles**

Message bundles for validation and localization:

```
messages.properties
messages_en.properties
messages_de.properties
```

Spring loads them automatically through `MessageSource`.

---

### **7. Framework metadata (META-INF)**

Advanced framework-level configuration:

```
src/main/resources/META-INF/
    spring.factories
    additional-spring-configuration-metadata.json
    spring-devtools.properties
```

Used for:

* auto-configuration
* metadata for IDE property hints
* customizing Spring Boot devtools
* building Spring Boot starters

---

### **8. Custom config/data files**

Optional small files required at runtime:

```
schema/
contracts/
examples/
```

Used for:

* event schemas (Avro/JSON Schema)
* sample payloads
* configuration templates
* small JSON/YAML files loaded manually

These must remain lightweight — large files should live outside the JAR.

---

## What does **not** belong here

* Secrets (API keys, passwords, tokens)
* Large binary files
* Logs or temporary files
* Uploaded content
* Build outputs
* Docker/Kubernetes files
* Project documentation not required at runtime

If a file changes per environment or contains sensitive data, keep it **outside** the JAR and load it via external config locations.

---

## Test resources

Tests have their own classpath:

```
src/test/resources/
```

Files here override main resources during tests.

Useful for:

* `application-test.yml`
* test data sets
* in-memory DB setup
* mock schemas

---

## How Spring uses resources

* All files become part of the packaged application under `BOOT-INF/classes/`.
* Spring loads configuration automatically based on name/order.
* Static files under `/static/**` are served without code.
* Flyway scans `db/migration/` and applies versioned scripts.
* Logging config is picked up if present at root.
* Message bundles are discovered automatically.

It is the central runtime asset directory for Spring Boot applications.

---

## Sub-cheatsheets (see child pages)

```
resources/
├─ app-config/
├─ db-migrations/
├─ logging/
├─ static-assets/
├─ i18n/
└─ meta-inf/
```

Each folder contains deeper, focused notes for that resource type.

## So what’s the *big story* of `resources/`?

It’s not a YAML folder.

It’s your **application’s runtime universe**:
the immutable stuff that ships with your code,
loaded from the classpath,
readable everywhere inside your app,
and structured enough for Spring to do smart things with it.

Once you see it like that, you see your cheatsheets naturally forming around the different tribes inside this universe:

1. **Config binding**
2. **Database migrations**
3. **Logging**
4. **Static assets / problem docs**
5. **i18n resources**
6. **Advanced metadata (META-INF)**

Those six topics are your future “properties + resources” cluster.

---

## Next up

✅ `app-config/overview.md`
or
✅ `db-migrations/flyway-basics.md`

