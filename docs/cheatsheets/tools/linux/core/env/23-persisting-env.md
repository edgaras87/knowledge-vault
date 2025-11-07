---
title: Persisting
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - shell
    - bash
    - cheatsheet
summary: A quick reference on how to persist environment variables across shell sessions in Linux by placing them in the appropriate startup files.
aliases:
    - Linux Environment Variables - Persisting Environment Variables Cheatsheet
---


---

# **Persisting Environment Variables**

Most environment variables only live inside a running shell. Close the terminal → they vanish.
Persistence means placing variables in the right startup file so they are loaded automatically for future sessions.

Linux has multiple startup layers. The key is understanding the rule:

**Each shell and each session type reads a different file.**

This page maps which file to use depending on what you want to persist.

---

## **The pyramid of persistence**

From narrow to wide:

1. **Current shell only**
   Defined manually or exported — disappears when you close the terminal.

2. **Interactive non-login shells**
   Loaded from `~/.bashrc`.

3. **Interactive login shells**
   Loaded from `~/.profile`, `~/.bash_profile`, or `~/.bash_login`.

4. **System-wide login environment**
   Loaded from `/etc/environment`.

5. **Scripted environments**
   Loaded from `/etc/profile` or `/etc/profile.d/*.sh`.

6. **Service environments (systemd)**
   Loaded from unit files — covered in `systemd-env.md`.

Which one you choose changes who receives the variable.

---

## **1. Persisting for interactive shells: `~/.bashrc`**

If you want every new terminal window (interactive shell) to have a variable:

```
# ~/.bashrc
export APP_TIMEOUT=10s
export PATH="$HOME/bin:$PATH"
```

After saving, reload:

```
source ~/.bashrc
```

Every new terminal gets it.

**Use this for:** developer tools, PATH changes, helper values.

---

## **2. Persisting for login shells: `~/.profile`**

A login shell is the first shell after you log in (SSH, console login, or some terminal emulators depending on settings).

For variables required on first login:

```
# ~/.profile
export LANG=en_US.UTF-8
export EDITOR=nvim
```

Reload:

```
source ~/.profile
```

**Use this for:** locale, editor preference, anything tied to your login identity.

---

## **3. System-wide login environment: `/etc/environment`**

This file is a *pure KEY=value file* — no `export`, no shell syntax.

Example:

```
JAVA_HOME="/usr/lib/jvm/java-21-openjdk"
APP_ENV="production"
```

This affects *all* users and *all* login sessions.

**Use this for:** system-wide defaults.

**Important:**
This is *not* a shell script. Do not use `$PATH` expansion or quotes with variables.

Correct:

```
PATH="/usr/local/bin:/usr/bin:/bin"
```

Incorrect:

```
PATH="$PATH:/new/location"   # does not work in /etc/environment
```

---

## **4. System profiles: `/etc/profile` and `/etc/profile.d/*.sh`**

These are shell scripts executed for login shells.

Example (system-wide):

```
# /etc/profile.d/myvars.sh
export APP_COLOR=blue
```

This is the cleanest place for administrators to define global shell-ready variables.

**Use this for:** controlled, scripted system-wide settings.

---

## **5. Variables for scripts only**

If you want scripts to use variables without forcing them on users:

* define inside the script
* or source a file inside the script

Example:

```
#!/bin/bash
source /opt/myapp/env.sh
python app.py
```

This keeps the variable scoped to execution, not to user sessions.

---

## **6. Making environment permanent for services (systemd)**

Systemd does not read your shell files.
Putting exports into `~/.bashrc` or `/etc/profile` has **no effect** on systemd services.

Service-specific persistence is done in:

```
/etc/systemd/system/myapp.service.d/env.conf
```

And inside:

```
[Service]
Environment="FOO=123"
EnvironmentFile=/etc/myapp/env
```

Details in `systemd-env.md`.

---

## **How to check which startup files your shell loads**

Print shell info:

```
echo $0
```

If it’s bash, check if it is a login shell:

```
shopt login_shell
```

Many terminal emulators start *non-login shells* (meaning they load only `.bashrc`).

---

## **Good patterns**

* Keep personal variables in `~/.bashrc`.
* Keep session-wide identity things in `~/.profile`.
* Use `/etc/environment` for global, simple settings.
* Use `/etc/profile.d/*.sh` for global scripted settings.
* Do not mix shell syntax into `/etc/environment`.
* Do not expect systemd services to read any of these.

