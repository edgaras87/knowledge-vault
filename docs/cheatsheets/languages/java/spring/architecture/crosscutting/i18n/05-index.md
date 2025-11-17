---
title: Content
date: 2025-11-10
tags: 
    - spring 
    - spring-boot
    - architecture
    - cheatsheet
      
summary: Tiny, framework-light helpers for working with time. This folder holds APIs (qualifier, facade, providers) that depend only on java.time. All wiring and property knobs live in config (see ClockConfig, TimeProps, …).
aliases:
    - Spring i18n Layer - Content Cheatsheet
---

# `i18n/` — Internationalization Utilities Index

Tiny, sharp tools for messages and problem texts.
APIs live here; wiring (MessageSource, injection processor) stays in `config/`.

---

## What lives here

* **`Messages`** — small facade over `MessageSource` with sane defaults.

  * `msg("key", args…)`, `tryMsg(...)`, `msgOrDefault(...)`, `problemTitle/detail(...)`.

* **`MessageCodes`** — constants + enums for type-safe keys.

  * `MessageCodes.App.GREETING`, `MessageCodes.Validation.*`,
    `MessageCodes.ProblemKey` (or standalone `ProblemKey` enum).

* **`ProblemKey`** *(if you chose standalone enum)* — the single source of truth for problem **slugs**, **title/detail keys**, and optionally **default HTTP status**.

* **`Message` (annotation)** — qualifier to mark fields for localized string injection.

* **`LocalizedMessage`** — functional interface: `render(args…)` returns a localized string using the current request locale.

> Wiring that pairs with this lives in `config/`:
> `MessageSourceConfig` (loads bundles) and `MessageInjectionPostProcessor` (injects `@Message` → `LocalizedMessage`).

---

## File tree (recommended)

```text
src/main/java/com/example/app/i18n/
├─ Messages.java            # Facade over MessageSource (LocaleContextHolder-aware)
├─ MessageCodes.java        # Constants + ProblemKey enum (or keep enum standalone below)
├─ ProblemKey.java          # (optional standalone) slug + keys [+ default status]
├─ Message.java             # @Message qualifier for field injection
└─ LocalizedMessage.java    # render(Object... args)
```

---

## Where bundles live (resources)

```text
src/main/resources/i18n/
├─ messages.properties              # app texts
├─ messages_de.properties           # localized variants
├─ problem-messages.properties      # RFC-7807 titles/details
└─ validation.properties            # (optional) Bean Validation overrides
```

---

## How it’s wired (reference)

```java
// config/MessageSourceConfig.java — loads basenames, UTF-8, no system-locale fallback
@Bean MessageSource messageSource() { ... }

// config/MessageInjectionPostProcessor.java — turns @Message fields into LocalizedMessage
@Configuration class MessageInjectionPostProcessor implements BeanPostProcessor { ... }
```

---

## Quick usage

### 1) Facade-based (constructor-friendly)

```java
@Service
class Greeter {
  private final Messages msg;
  Greeter(Messages msg) { this.msg = msg; }

  String greet(String name) {
    return msg.msg(MessageCodes.App.GREETING, name); // "Hello, Edgaras!"
  }
}
```

### 2) Field injection with `@Message`

```java
@RestControllerAdvice
class Errors {
  @Message("problem.duplicate-category.title") LocalizedMessage dupTitle;
  @Message("problem.duplicate-category.detail") LocalizedMessage dupDetail;

  // pd.setTitle(dupTitle.render()); pd.setDetail(dupDetail.render(name));
}
```

### 3) Enum-backed problems

```java
var key = ProblemKey.CATEGORY_NOT_FOUND;
var title  = msg.msg(key.titleKey());
var detail = msg.msg(key.detailKey(), id); // args fill {0}, {1}, …
```

---

## Conventions

* **Keys are stable contracts.** Favor short, dot-separated names: `app.greeting`, `problem.duplicate-category.title`.
* **Problem slugs are URL-friendly** (`kebab-case`) and drive both keys and `/problems/{slug}` docs.
* **Locale source:** `LocaleContextHolder` (set by `WebConfig`’s Accept-Language resolver).
* **Missing keys:** return the code itself (via `useCodeAsDefaultMessage=true`). You can flip to strict in prod if desired.

---

## Gotchas (and fixes)

* **Pluralization:** simple two-form with pipes in text (choose in code), or add ICU MessageFormat if you need full plural rules.
* **Escaping:** single quotes in bundles must be doubled (`don''t`).
* **Validation messages:** keep in `validation.properties` or mix into `messages.properties`—just be consistent.

---

## Checklist

* `Messages` bean exposed via `MessagesConfig` (or created directly where needed)
* `MessageSourceConfig` loads `i18n/messages`, `i18n/problem-messages`, `i18n/validation`
* `MessageInjectionPostProcessor` active (if using `@Message`)
* `ProblemKey` slugs and bundles in sync
* `/problems` + `/problems/{slug}` controllers (optional docs) read titles from the same keys

---

## See also

* `web/support/errors/ProblemTexts` — builds RFC-7807 from `ProblemKey` + `Messages`
* `web/support/links/ProblemLinks` — canonical `/problems/{slug}` URIs
* `config/WebConfig` — locale resolver + validator bound to `MessageSource`
