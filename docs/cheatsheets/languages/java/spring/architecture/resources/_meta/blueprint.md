---
title: Resources Future Blueprint
date: 2025-11-06
summary: A blueprint for organizing a comprehensive cheat sheet section on the `src/main/resources/` directory in Spring applications.
---

# Spring Architecture Cheat Sheet: `src/main/resources/`

Let’s carve out a clean, future-proof **`cheatsheets/resources/`** section for your knowledge vault.
Think of this as mapping the underground chambers of `src/main/resources/`: not just what they contain, but *why* they exist, how Spring uses them, and what you’ll need as your projects grow.

No lists for the sake of lists — each directory is tied to a real architectural role.

The tone stays conversational; the structure stays crisp.

---

## The shape of the territory

Your project’s `resources/` directory is like the backstage of a theater.
Configuration, scripts, templates, docs — all the inanimate stuff needed for the play.

A good cheatsheet cluster mirrors that.

Here’s the best structure to adopt now:

```
cheatsheets/resources/
├─ index.md
├─ app-config/
│  ├─ overview.md
│  ├─ yaml-vs-properties.md
│  ├─ application-profiles.md
│  ├─ config-binding.md              # @ConfigurationProperties patterns
│  └─ overrides-and-env.md           # env vars, external files, precedence
├─ db-migrations/
│  ├─ flyway-basics.md
│  ├─ migration-patterns.md
│  └─ testing-migrations.md
├─ logging/
│  ├─ logback-config.md
│  ├─ pattern-layouts.md
│  └─ environment-levels.md
├─ static-assets/
│  ├─ serving-static.md
│  ├─ problem-docs.md
│  └─ api-docs-static.md
├─ i18n/
│  ├─ messages-basics.md
│  ├─ validation-messages.md
│  └─ locale-switching.md
└─ meta-inf/
   ├─ spring-factories.md
   ├─ auto-config-metadata.md
   └─ devtools-config.md
```

Let’s walk through what each chamber is for, and why it’s worth having its own cheatsheet.

---

## `index.md` — the “what lives in resources” map

This is the high-level overview you can read in 30 seconds.
It explains:

* The purpose of `resources/`
* Which subfolders are typical
* When to use resources vs code vs environment

This is the quick mental compass.

---

## `app-config/` — your main habitat

This folder houses everything around:

* `application.yml`
* profiles
* YAML structure
* property sources
* typed binding
* overrides
* environment-specific config

You already have solid property/binding notes; extracting them into smaller, focused pages makes them easier to search and reference.

Most important topics:

**`config-binding.md`**
Your excellent `@ConfigurationProperties` knowledge belongs here.

**`application-profiles.md`**
Dev / test / prod behavior toggles.

**`overrides-and-env.md`**
Where values *actually* come from in production:
container env vars, `spring.config.import`, `.env`, systemd overrides, etc.

This folder becomes your “config brain”.

---

## `db-migrations/` — where data evolution is documented

Soon you’ll create entities with relations.
Schema changes will become a daily ritual.
This folder is where your future self thanks you.

You want:

**`flyway-basics.md`**
Versioned vs repeatable migrations, naming (`V1__init.sql`).
Folder structure (`resources/db/migration/`).
How Spring runs them.

**`migration-patterns.md`**
Multi-entity schema evolutions.
Foreign keys.
Indexing.
Rollback strategies.
When to split migrations.

**`testing-migrations.md`**
How to test migrations with Testcontainers.

This ties directly into your long-term goals (bigger example project with many entities).

---

## `logging/` — the “stop guessing what the logs do” section

A Spring app with real production viability needs:

* Correct log levels per environment
* A sane pattern layout
* Request correlation IDs (MDC)
* JSON logging (if later needed)

These docs cover:

**`logback-config.md`**
Where to put logback files (`logback-spring.xml`).
How Spring loads them.
How profiles change logging.

**`pattern-layouts.md`**
Human-readable vs JSON logs.
Correlation IDs.
Thread info.
Why structured logs matter.

**`environment-levels.md`**
DEBUG in dev, INFO in prod.
Per-package log levels.

This fills a real gap in your current cheatsheets — and you’ll need it soon.

---

## `static-assets/` — the quiet but crucial part

Even if you’re backend-only, Spring serves static files from:

```
src/main/resources/static/
```

This enables:

* API docs hosted under `/docs/*`
* ProblemDetail definition pages (`/problems/*`)
* A health landing page
* Privacy policy/static pages

Your cheatsheets:

**`serving-static.md`**
How Spring Boot serves static content.
How caching works.
MIME types.

**`problem-docs.md`**
You’ll tie this to your ProblemDetail URIs:

```
problems/
    invalid-input.html
    duplicate-category.html
```

Perfect for your growing RFC-7807 setup.

---

## `i18n/` — when your API grows up

Even for REST APIs:

* Validation messages live here
* Domain errors often want localized templates
* UI clients rely on i18n even if backend is headless

`messages-basics.md` helps you:

* Understand resource bundles
* Keys vs fallback locales
* How Spring loads message sources

`validation-messages.md` ties into your Bean Validation notes:

```
@NotBlank(message = "{user.email.required}")
```

Connected to:
`src/main/resources/messages.properties`.

---

## `meta-inf/` — the arcane magic spells

You won’t use this daily, but it’s essential once you start:

* auto-configuring your own modules
* building reusable starters
* customizing devtools reload behavior
* overriding Spring Boot metadata

The cheatsheets introduce:

**`spring-factories.md`**
How Spring Boot discovers your auto-configurations.

**`auto-config-metadata.md`**
Describe your properties to IDEs (so they autocomplete).

**`devtools-config.md`**
Control hot reload, exclude certain paths from restart.

You don’t need all this immediately — but you *will* when building more advanced Spring infrastructure.

---

## Why this structure works

It matches the real-world growth of a Spring Boot developer:

1. *start with properties*
2. *grow into profiles*
3. *then migrations*
4. *then logging*
5. *then static docs*
6. *eventually auto-config metadata*

Everything under `resources/` matches one of these domains.

You won’t drown in a giant monolithic cheatsheet.
You’ll open exactly the page you need, based on what you’re working on today.

---

## If you want next:

I can generate **any sub-cheatsheet** in this tree in your preferred style (like the long properties one you have).

The natural next step is:

⚡ **`db-migrations/flyway-basics.md` — Flyway for multi-entity projects**
Because your next practice goals involve multiple related entities.

Or we can build **the `resources/index.md`** first to anchor the whole section.

You choose the next stone to place, and the rest will naturally fit around it.
