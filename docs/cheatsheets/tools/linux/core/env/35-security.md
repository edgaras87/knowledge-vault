---
title: Security
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - security
    - cheatsheet
summary: A quick reference on the security implications of using environment variables for sensitive data in Linux, including potential leak vectors and safer alternatives.
aliases:
    - Linux Environment Variables - Security Cheatsheet
---

# **Environment Variable Security**

Environment variables feel convenient for secrets — but convenience hides sharp edges. These variables are attached to running processes, exposed through `/proc`, sometimes captured in logs, sometimes dumped during crashes, and occasionally inherited in ways you didn’t expect.

They are not “dangerous,” but they’re not a secure storage mechanism. This page gives you the real boundaries so you can choose wisely.

---

## **The uncomfortable truth**

Environment variables were never designed for secrets.
They are plaintext strings attached to processes. Anyone who can examine or debug the process may see them.

They are safe enough for:

* feature flags
* mode toggles
* configuration (paths, regions, email hosts)

They are **risky** for:

* database passwords
* API tokens
* encryption keys
* anything that breaks trust if leaked

Not forbidden — just fragile.

---

## **Where environment variables leak**

### 1) **`/proc/<pid>/environ`**

Any user who can read the process’s `/proc/<pid>` directory can read the full environment.

Example:

```
tr '\0' '\n' < /proc/1234/environ
```

If the service runs as a dedicated user and file permissions are tight, exposure is limited — but still plaintext.

---

### 2) **Crash dumps / core dumps**

If the process crashes and core dumps are enabled, the environment is stored inside the dump.

Developers reading the dump can see secrets.

---

### 3) **Process listings (`ps`)**

Some process inspectors show environment variables:

```
ps eww 1234
```

Systemd hides some variables for some services, but many are visible.

---

### 4) **Debug logs or accidental printing**

If your app logs its environment (common during debugging), secrets end up in logs.

Logs live for years.
Logs get copied.
Logs get emailed.
Logs get shipped to analytics.

Never trust logs.

---

### 5) **Shell history contamination**

If you do:

```
export DB_PASSWORD=supersecret
```

that’s fine.

But if you run commands like:

```
DB_PASSWORD=supersecret ./myapp
```

some shells store it in history. Others scrub, but you can’t depend on it.

---

### 6) **Child processes**

Environment variables flow down:

```
parent → child → grandchild …
```

If a child program logs its environment (or passes it further), the secret propagates beyond your control.

Containers do the same: env vars flow into PID 1 inside the container unless filtered.

---

## **Systemd’s stance on secrets**

Modern systemd includes a **credential system** designed specifically to avoid environment leakage.

### Prefer this:

**Unit file:**

```ini
[Service]
LoadCredential=DB_PASSWORD:/etc/secure/db_password
```

Systemd will mount a read-only file at:

```
/run/credentials/myapp.service/DB_PASSWORD
```

Your app reads from a file — not env.

No exposure through `/proc/<pid>/environ`.
No exposure via `ps`.
No inheritance.
No accidental expansions.
And systemd erases it when the service stops.

This is the right way to supply secrets to systemd services.

---

## **Safer alternatives for secrets**

### 1) **Files with strict permissions**

```
chmod 600 /etc/myapp/secret
chown myapp:myapp /etc/myapp/secret
```

App reads the file. No environment leaks.

### 2) **systemd credentials** (preferred)

As shown above.

### 3) **Password stores / secret managers**

Vault, sops/age, AWS Secrets Manager, Kubernetes Secrets, GCP Secret Manager.

Not always necessary locally, but excellent practice for production.

### 4) **Kernel keyring**

Advanced, rarely used.
Secure per-session keyring API.

---

## **When environment variables are still acceptable**

Sometimes you *can* use env vars, if you understand the boundaries.

Valid cases:

* local development
* docker-compose projects
* low-risk apps
* toggles (`FEATURE_X=true`)
* no secrets, just configuration

Just don’t treat environment variables as encrypted vaults. They’re not.

---

## **Minimal hardening when you *must* use env vars**

### 1) Lock down access to the process owner

Make sure the service runs as a dedicated, unprivileged user:

```ini
[Service]
User=myapp
Group=myapp
```

### 2) Protect env files

```
chmod 600 /etc/myapp/myapp.env
chown myapp:myapp /etc/myapp/myapp.env
```

### 3) Don’t log the environment

Disable debug dumps that print env.

### 4) Review `ps` visibility

Check what your process reveals:

```
ps eww $(pidof myapp)
```

### 5) Avoid passing secrets via command line

Never do:

```
pass=secret ./app
```

Many tools capture command lines (`ps`, logs, auditd).

---

## **Mental model (short and correct)**

* Environment variables are **visible**.
* Environment variables are **inherited**.
* Environment variables are **persistently stored in memory** as plaintext.
* Environment variables **do not protect** secrets.
* They are fine for flags, not fine for keys.

To store secrets securely, use files with restricted permissions or systemd credentials. Always assume env values can leak through introspection.

