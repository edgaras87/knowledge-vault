---
title: Systemd Services
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - systemd
    - cheatsheet
summary: A quick reference on how to set and manage environment variables for systemd services, including inline settings, external files, and best practices.
aliases:
    - Linux Environment Variables - Environment for Systemd Services Cheatsheet
---



# **Environment for systemd services**

Systemd does **not** read your shell files. Services get env vars only from:

* the unit file (`Environment=…`)
* environment files (`EnvironmentFile=…`)
* the manager’s own environment (`PassEnvironment=…`)
* system-wide defaults (`DefaultEnvironment=` in `systemd-system.conf`)
* container/orchestrator injection (when PID 1 is systemd inside a container)

Everything else (like `~/.bashrc`) is ignored.

---

## **Quick start (minimal, correct)**

Create/override a unit safely:

```bash
sudo systemctl edit myapp.service
```

Paste:

```ini
[Service]
Environment="APP_MODE=prod" "PORT=8080"
# Optional env file (KEY=VAL per line)
EnvironmentFile=-/etc/myapp/myapp.env
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

Verify:

```bash
systemctl show myapp.service -p Environment
# Or the truth serum:
pid=$(systemctl show myapp.service -p MainPID --value)
tr '\0' '\n' < /proc/$pid/environ | sort
```

---

## **`Environment=` — inline assignments**

* Accepts one or more `KEY=VAL` entries.
* Quote if value contains spaces or `#`.

Examples:

```ini
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk"
Environment="JAVA_OPTS=-Xms256m -Xmx512m -Duser.timezone=UTC"
Environment="APP_FLAGS=--color=always --port=8080"
```

> Systemd expands environment variables in command lines (`ExecStart=`), so `$APP_FLAGS` is usable there.

---

## **`EnvironmentFile=` — external file(s)**

* Format: one `KEY=VAL` per line. `#` starts a comment. No shell logic.
* Use a leading `-` to ignore missing files.
* You can include multiple files; later ones can override earlier ones.

Example file `/etc/myapp/myapp.env`:

```
APP_MODE=production
PORT=8080
# Quotes are okay when you need spaces:
APP_FLAGS="--color=always --port=8080"
```

Unit snippet:

```ini
[Service]
EnvironmentFile=/etc/myapp/myapp.env
EnvironmentFile=-/etc/myapp/extra.env
```

---

## **Using env vars in `ExecStart=`**

Systemd expands `$VAR` and `${VAR}` from the service environment:

```ini
[Service]
Environment="APP_FLAGS=--port=9000 --color=always"
ExecStart=/usr/local/bin/myapp $APP_FLAGS
```

> This is **not a shell**: redirection, pipes, `$(…)`, `~` expansion do **not** work unless you explicitly run a shell:
>
> ```ini
> ExecStart=/bin/bash -lc '/usr/local/bin/myapp $APP_FLAGS | tee -a /var/log/my.log'
> ```
>
> Prefer direct exec without shells unless you truly need shell features.

---

## **`PassEnvironment=` — inherit from the manager**

Sometimes you want the unit to inherit variables that live in the systemd **manager** (not your shell). Use:

```bash
# Set vars into the system instance manager (survive until reboot or unset)
sudo systemctl set-environment REGISTRY_MIRROR=https://mirror.local FEATURE_X=true

# See what's set:
systemctl show-environment | sort
```

Then in your unit:

```ini
[Service]
PassEnvironment=REGISTRY_MIRROR FEATURE_X
```

> Useful for quick, cross-service toggles. Remove with `sudo systemctl unset-environment REGISTRY_MIRROR FEATURE_X`.

---

## **Drop-ins & overrides (the right way to customize)**

* Never edit vendor files in `/lib/systemd/system/`.
* Use drop-ins in `/etc/systemd/system/<name>.service.d/*.conf` created via `systemctl edit`.
* `override.conf` beats vendor defaults, survives package upgrades.

---

## **Debian vs RHEL traditions**

* Debian/Ubuntu often use `/etc/default/<name>` as an env file (plain `KEY=VAL`).
* RHEL/CentOS/Fedora often use `/etc/sysconfig/<name>`.
* Both patterns map cleanly to `EnvironmentFile=`.

---

## **Global defaults (rare, but exists)**

`/etc/systemd/system.conf` (system instance) or `/etc/systemd/user.conf` (user instance):

```ini
[Manager]
DefaultEnvironment="LC_ALL=C.UTF-8" "HTTP_PROXY=http://proxy:3128"
```

Then `daemon-reload`. These become the **manager’s** environment—units still must opt-in with `PassEnvironment=` or have their own `Environment=`.

---

## **User services vs system services**

* **System services**: `sudo systemctl [status|edit|restart] myapp.service`
* **User services** (no sudo): `systemctl --user edit myapp.service`

  * Persist drop-ins under `~/.config/systemd/user/`
  * Enable lingering if you want them outside active login sessions: `sudo loginctl enable-linger "$USER"`

Environments are separate per manager (system vs user).

---

## **Troubleshooting checklist**

1. **Did you reload & restart?**
   `sudo systemctl daemon-reload && sudo systemctl restart myapp.service`
2. **What does the process actually have?**
   `/proc/$pid/environ` beats assumptions.
3. **Spaces/quotes?**
   Use `Environment="KEY=value with spaces"`.
4. **Shell features in `ExecStart`?**
   They won’t work unless you call a shell explicitly.
5. **`sudo` vs non-sudo?**
   System manager env differs from your shell; `PassEnvironment=` only sees what the manager knows.
6. **Wrong file type?**
   `/etc/environment` is not read by systemd services. Use `EnvironmentFile=`.

---

## **Security notes (don’t feed secrets to hungry logs)**

* Environment variables are visible via `/proc/<pid>/environ` to users with sufficient permissions, and can leak into crash dumps or debug logs.
* Prefer **systemd credentials** for real secrets (supported on modern systemd):

  * In the unit:

    ```ini
    [Service]
    LoadCredential=DB_PASSWORD:/etc/secure/db_password
    ```
  * The service will read it at runtime from `/run/credentials/%N/DB_PASSWORD` (file, not env).
* If you must use env for secrets, confine permissions on env files (`chmod 600`) and restrict who can `systemctl status` or read `/proc/<pid>/environ`.

---

## **Patterns you’ll reuse**

**Single file, clean handoff**

```ini
[Service]
EnvironmentFile=/etc/myapp/myapp.env
ExecStart=/usr/local/bin/myapp $APP_FLAGS
```

**Per-host overlay (optional file)**

```ini
[Service]
EnvironmentFile=/etc/myapp/common.env
EnvironmentFile=-/etc/myapp/host-overrides.env
```

**Feature flags from manager**

```ini
# One-time setup
sudo systemctl set-environment FEATURE_BETA=true

# Unit
[Service]
PassEnvironment=FEATURE_BETA
```

