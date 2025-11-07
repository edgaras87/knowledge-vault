---
title: Naming Conventions
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - shell
    - bash
    - cheatsheet
summary: A quick reference on best practices and conventions for naming environment variables in Linux to ensure clarity, consistency, and compatibility across different systems and applications.
aliases:
    - Linux Environment Variables - Naming Conventions Cheatsheet
---

# **Environment Variable Naming Conventions**

Environment variables are primitive: all you get is `KEY=value`.
Good naming is your only structure. A predictable naming scheme prevents confusion, collisions, and broken behavior in shells, systemd, Docker, and frameworks like Spring.

This page gives you a stable, future-proof pattern.

---

## **General rules**

### **1) Always UPPERCASE**

Environment variables are conventionally uppercase.

```
APP_ENV
APP_MODE
DATABASE_URL
API_TOKEN
```

Linux is case-sensitive, so this prevents accidental clashes with shell variables or functions.

---

### **2) Use snake_case with underscores**

Words separated by `_`:

```
APP_FEATURE_SIGNUP
APP_TIMEOUT_SECONDS
LOG_LEVEL
```

Underscores are universally safe — shells, systemd, Docker, Kubernetes all accept them.

---

### **3) Never use spaces or special characters**

Avoid:

* spaces
* quotes
* punctuation
* dashes (`-`)
* dots (`.`)

Bad:

```
app-mode=prod
app.mode=prod
APP-MODE=prod
```

Good:

```
APP_MODE=prod
```

Dots are guaranteed trouble: shell syntax, Spring binding, Kubernetes, systemd — all treat dots differently.

---

## **Prefixing: isolate domains**

Prefix variables by subsystem, service, or app.

Examples:

```
APP_DB_URL
APP_DB_USER
APP_DB_POOL_SIZE
```

or:

```
PAYMENTS_API_URL
PAYMENTS_API_KEY
```

or systematically per service:

```
SHOPPINGCART_HOST
SHOPPINGCART_PORT
SHOPPINGCART_JWT_SECRET
```

Prefixing prevents collisions when multiple apps share a runtime environment.

---

## **Hierarchical naming (flat but structured)**

Env vars can’t nest, but you can simulate hierarchy:

```
APP_FEATURE_SIGNUP_ENABLED=true
APP_FEATURE_RECOMMENDATIONS_ENABLED=false
APP_LIMITS_UPLOAD_SIZE_MB=20
```

A pattern emerges:

```
<APP>_<NAMESPACE>_<KEY>
```

This gives you structure without breaking compatibility.

---

## **Booleans and flags**

Use simple `true` / `false` or `1` / `0`.

```
FEATURE_X=true
DEBUG=1
```

Avoid weird values:

```
FEATURE_X=yes please   # breaks many tools
```

Stick to minimal, predictable values — your future self will thank you.

---

## **Numbers and sizes**

Be explicit:

```
APP_TIMEOUT_SECONDS=30
APP_RATE_LIMIT=200
APP_MAX_UPLOAD_MB=50
```

Avoid vague:

```
APP_TIMEOUT=fast
APP_SIZE=bigger
```

Let the consuming app convert units.

---

## **PATH-like lists**

List items separated by `:`:

```
APP_PLUGIN_PATH="/opt/app/plugins:/home/user/plugins"
```

This works with shells, systemd, and most POSIX tools.

If you need spaces, **quote the full value**, not individual pieces:

```
APP_PATHS="/opt/My App/bin:/opt/Other App/bin"
```

---

## **Secrets naming style**

If you *must* store secrets in env vars (understanding the risks described in `security.md`), name them clearly:

```
DB_PASSWORD
API_TOKEN
JWT_SECRET
SMTP_PASSWORD
```

This reduces ambiguity across services, but it doesn’t solve the leakage problem — only helps you keep track of what’s sensitive.

---

## **Spring-specific note (when interacting)**

Spring automatically binds environment variables to properties by converting `_` to `.`:

```
APP_DB_USER → app.db.user
APP_FEATURE_SIGNUP_ENABLED → app.feature.signup.enabled
```

This is why underscores are the best delimiter.
Dashes or dots break resolution or require escape rules.

---

## **Docker & Kubernetes conventions**

Both expect:

* uppercase
* underscores
* no dots
* no quotes (shell quoting removed)

Example:

```
docker run -e APP_MODE=prod -e DB_URL=postgres://...
```

Kubernetes `env:` keys follow same conventions.

---

## **Systemd conventions**

Systemd accepts:

* uppercase
* underscores
* quoted or unquoted values
* no shell expansions unless inside `bash -lc`

Good:

```
Environment="APP_FEATURE_LOGIN=true"
```

Bad:

```
Environment="app.mode=prod"
```

Dots will not break systemd, but they break multi-layer tooling (Spring, Bash expansion, shell scripts). Avoid.

---

## **Mental model**

Environment variables are the least structured configuration mechanism in Linux.
Your naming scheme *is* the structure.
Consistency beats cleverness.

A good name should answer immediately:

* which app it belongs to
* what domain it configures
* what type the value is
* whether it is safe or sensitive

