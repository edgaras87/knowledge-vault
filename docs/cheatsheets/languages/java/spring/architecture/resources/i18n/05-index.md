---
title: Content
date: 2025-11-07
tags: 
    - java
    - spring
    - i18n
    - internationalization
    - cheatsheet
summary: A quick reference on internationalization (i18n) in Spring applications, covering message bundles, locale resolution, and integration with Bean Validation.
aliases:
    - Spring i18n - Content Cheatsheet

---


# i18n in Spring — Index

The `i18n/` section of your resources cheatsheet is the library of **how your application speaks**.  
Not Java code speaking — but the strings that appear on the wire, in logs, or in validations.

Even for a backend-only API, this matters more than it first seems:  
error messages, validation messages, field-level hints, localized business rules… every system eventually leaks language.

Spring’s i18n story is deceptively simple: put message bundles under `resources/`, wire a message source, let the rest of the framework do the lifting.  
This index gives you the mental model before diving into subpages.

---

## Why i18n matters even for REST APIs

APIs look numeric and structured until they sink their teeth into the human layer:

- Bean Validation messages (e.g., `email.required`)
- Exception messages (domain, application, web)
- ProblemDetail `detail` and `title` fields
- Business rules that differ per locale or region
- Clients that need user-facing copy but expect backend to supply it

Internationalization becomes the **grammar of your errors**.  
This cheatsheet cluster helps you shape that grammar consciously.

---

## Where i18n lives in Spring projects

Spring loads message bundles from:

```

src/main/resources/
messages.properties
messages_en.properties
messages_de.properties
...

```

Cheatsheet folder mirrors that:

```
cheatsheets/resources/i18n/
├─ index.md                ← you are here
├─ messages-basics.md      ← how Spring loads and resolves bundles
├─ validation-messages.md  ← connecting Bean Validation to i18n
└─ locale-switching.md     ← how Spring picks the user’s language
```

Each file focuses on one architectural responsibility.

---

## The mental model (keep this one close)

A message lookup in Spring is a three-step dance:

1. **Key**: Something like `"category.name.required"` that represents meaning, not a specific language.

2. **Bundle**: A `.properties` file containing localized text.

3. **Locale**: The user’s language preference, resolved by Spring (HTTP headers, session, cookie, or custom strategy).

Spring’s `MessageSource` glues those parts:  
*give me key X in locale Y, fallback to default if missing*.

Once you internalize this, everything else is just tuning knobs.

---

## What each subpage covers

### `messages-basics.md` — the foundation

How messages files work, how fallback works, how Spring Boot auto-wires a default `MessageSource`, and how you customize `basename`, `encoding`, or caching.

This is the first stop if you’re trying to understand *why Spring isn’t picking up your translations*.

### `validation-messages.md` — where i18n meets Bean Validation  

You’ll use this constantly. Shows how to move from:

```java
@NotBlank(message = "Email is required")
```

to

```java
@NotBlank(message = "{user.email.required}")
```

…and how Spring finds that key in `messages.properties`.
Includes nested bundles, overriding, and per-validation-constraint i18n patterns.

### `locale-switching.md` — “how does the app know which language to speak?”

Describes the mechanisms:

* `Accept-Language` header
* session/cookie strategies
* custom resolvers
* URL-based locale selection

Useful when you grow into user-facing or multi-region APIs.

---

## How this index fits in your resources blueprint

This `i18n/` cluster corresponds to the “text layer” of your backend. It’s the sibling of:

* **exception-handling** (they often share message keys)
* **ProblemDetail docs** (static pages under `static/`)
* **validation cheatsheets**
* **configuration properties** (some apps externalize display names or units)

It becomes an anchor once your domain model produces meaningful, human-readable errors.

---

## Where to go next

The natural continuation is:

⚡ **`messages-basics.md`**
because everything else — validation, localized ProblemDetail, static docs — builds on that single idea:

keys → bundles → locale → text.

Once we shape that file, the rest click into place.
