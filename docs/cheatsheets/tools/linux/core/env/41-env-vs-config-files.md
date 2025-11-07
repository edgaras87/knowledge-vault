---
title: Env vs Config Files
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - config-files
    - cheatsheet
summary: A quick reference on when to use environment variables versus configuration files for application settings in Linux, including best practices and common scenarios.
aliases:
    - Linux Environment Variables - Environment Variables vs Config Files Cheatsheet
---



# **Environment Variables vs Config Files**

Environment variables are fast, simple, global, and ephemeral.
Config files are structured, explicit, and versionable.
Both solve configuration problems, but they fit different shapes.

This page gives you a clear, future-proof rulebook for choosing between them.

---

## **The short rule**

Use **environment variables** for *small, dynamic, deployment-time configuration*.
Use **config files** for *complex, structured, stable configuration*.

That’s it — but let’s expand it into the practical cases you’ll face.

---

## **When to use environment variables**

### **1) Deployment-time differences**

Things that change between machines or environments:

```
APP_ENV=prod
APP_DEBUG=false
APP_PORT=8080
```

These values stay close to the runtime context — perfect for CI/CD, Docker, systemd.

---

### **2) Secrets (only if you accept the risks)**

Environment variables are commonly used for credentials:

```
DB_PASSWORD=secret
API_TOKEN=xyz
```

But see `security.md`: env vars are not secure storage.
Prefer systemd credentials or files with strict permissions.

---

### **3) Feature flags**

Simple on/off toggles:

```
FEATURE_SIGNUP=true
FEATURE_CACHE_WARMUP=false
```

Flags work well as env vars because they’re tiny, atomic, and safe to override.

---

### **4) Ephemeral runtime overrides**

Temporary adjustments:

```
APP_TIMEOUT=5s ./myapp
```

No persistence, no cleanup.

---

### **5) Containerized apps (Docker, Podman, K8s)**

Containers are built for env-based configuration:

```
docker run -e APP_MODE=prod -e LOG_LEVEL=debug myapp
```

Kubernetes `env:` and `envFrom:` integrate smoothly with env-shaped config.

---

## **When NOT to use environment variables**

### **1) Complex or structured configuration**

Environment variables are flat; any hierarchy becomes ugly:

```
APP_DB_HOST=...
APP_DB_PORT=...
APP_DB_POOL_MIN=...
APP_DB_POOL_MAX=...
```

A config file is cleaner:

```yaml
db:
  host: localhost
  port: 5432
  pool:
    min: 4
    max: 20
```

If the structure matters, use a file.

---

### **2) Large configuration blobs**

Don’t do this:

```
APP_RULESET_JSON='{"rules":[...]}'
```

Multi-line strings, JSON, XML, SQL, PEM keys — all painful in environment variables.

Config files are safer and more maintainable.

---

### **3) Secrets that need real protection**

Environment variables:

* leak via `/proc/<pid>/environ`
* appear in core dumps
* can leak to logs
* propagate to child processes

**Use:**

* files with `chmod 600`,
* systemd credentials,
* secret managers.

---

### **4) Anything that should be version-controlled**

Environment variables live outside source control.
If config is part of the application’s definition, put it in a file.

Good:

```
config/default.yml
config/production.yml
```

Bad:

```
APP_COLOR_SCHEME=galactic   # no one will remember this later
```

---

### **5) Platform-specific complex configuration**

Cron, systemd, SSH, container runtime — each environment behaves differently.
Config files provide consistency across platforms.

---

## **Hybrid approach (best practice for stable apps)**

A powerful pattern:

**Config file as base → environment variables override it.**

Example:

`/etc/myapp/config.yml`:

```yaml
db:
  host: localhost
  port: 5432
  user: service
  pool:
    max: 20
```

Env at runtime:

```
DB_HOST=prod-db.company.net
```

The app starts with defaults from the file, then applies env overrides.

This hybrid setup gives:

* structure
* clarity
* easy overrides
* safe defaults

Spring Boot, Django, Rails, and Node ecosystems all support this model.

---

## **Systemd-specific choice**

### **Environment variables**

Good for:

* PORT
* LOG_LEVEL
* APP_ENV
* toggles
* tiny secrets (temporary)

### **EnvironmentFile=**

Good for:

* small, flat, multi-variable configurations
* system-wide deployment management

### **LoadCredential= (recommended for sensitive data)**

Good for:

* secrets (passwords, tokens)
* files containing keys
* anything that should not appear in `/proc/<pid>/environ`

### **Config files**

Good for:

* full application configuration
* complex YAML/JSON structures
* settings shared across environments

---

## **Containers and envs (modern reality)**

In Docker/Compose/K8s worlds, env vars dominate because:

* simple injection
* easy to override in CI
* no file mounting required
* stateless design encourages flat config

But even then, for sensitive or structured config, containers use:

* mounted config files
* mounted secret volumes
* encrypted secrets (sealed-secrets, Vault, SOPS)
* ConfigMaps (K8s)

The pattern doesn’t change — env vars are for the “small stuff.”

---

## **Mental model**

Use env vars when you want:

* simple
* dynamic
* flat
* small
* injected at runtime

Use config files when you need:

* structure
* safety
* version control
* complexity
* long-term maintainability

Think of env vars as toggles; config files as blueprints.

