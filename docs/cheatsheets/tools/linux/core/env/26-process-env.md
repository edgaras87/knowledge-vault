---
title: Process 
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - shell
    - bash
    - cheatsheet
summary: A quick reference on how process environment variable inheritance works in Linux, including how to inspect a running process's environment and common pitfalls.
aliases:
    - Linux Environment Variables - Process Environment Variables Cheatsheet
---

# **Process Environment — Inheritance & Inspection**

The environment is copied at process birth. A parent gives the child a snapshot (`KEY=value` pairs). After that moment, they’re separate worlds.

This page shows how inheritance really works and how to *observe* a running process’s environment (your best debugging tool).

---

## **Lifecycle in one line**

**fork → (optional mutations) → execve(env passed)**

* `fork` duplicates the parent (including env).
* The child may tweak its env.
* `execve` replaces the program; the env snapshot you pass is what the new process starts with.
* After start, **you cannot change a child’s environment** from outside.

---

## **Core rules**

1. **Copy-on-start:** Child inherits a *copy* of the parent’s environment.
2. **No global store:** There is no “system environment.” Only per-process environments.
3. **One-way street:** Parents can shape children; children can’t retroactively change parents.
4. **Export matters:** Only exported variables enter the environment and get inherited.
5. **Services ≠ shells:** systemd, cron, containers each craft their own starting environment.

---

## **See what a process actually has**

You don’t know until you look. On Linux, each process exposes its env at `/proc/<pid>/environ` (NUL-separated).

### Pretty-print a PID’s environment

```bash
# Replace 1234 with your PID
tr '\0' '\n' < /proc/1234/environ
```

### Your own process (the shell running this command)

```bash
tr '\0' '\n' < /proc/$$/environ
```

### Using `strings` (quick & dirty)

```bash
strings -0 /proc/1234/environ
```

### `ps` quick glance

```bash
ps eww 1234            # shows env appended to the command (long line)
ps e -o pid,cmd 1234   # compact, still includes env
```

> Tip: If you see nothing, you might lack permissions or the process may have been sanitized (e.g., via `sudo` env cleaning, container runtime, or `systemd` restrictions).

---

## **Reproducing a child’s view**

If a child sees something surprising, emulate *its* parent context.

### Launch with extra env

```bash
FOO=42 BAR=dev myapp
```

### Launch with a **clean** env (great for PATH bugs)

```bash
env -i PATH=/usr/bin:/bin bash -lc 'printenv'
```

### Watch the env passed at exec (forensics)

```bash
strace -e execve -f myapp 2>&1 | grep -A2 execve
```

You’ll see `execve("/path/to/myapp", [...], ["KEY=VAL","..."])` — the exact env snapshot.

---

## **Common inheritance traps**

### 1) “Works in my shell, fails as a service”

* Your shell loads `~/.bashrc`, `/etc/profile`, etc.
* **systemd doesn’t**. It builds its own environment.
* Fix in the unit: `Environment=KEY=VAL` or `EnvironmentFile=/etc/myapp/env`.
  (Details in `systemd-env.md`.)

### 2) `sudo` dropped my variables

* Defaults often sanitize env (`env_reset`).
* Options like `sudo -E` preserve a curated set; whitelist with `/etc/sudoers` `env_keep`.
* Treat `sudo` sessions as a different parent.

### 3) `cron` jobs can’t find my PATH

* Cron has a tiny, minimal environment.
* Set what you need at top of crontab or script:

  ```
  PATH=/usr/local/bin:/usr/bin:/bin
  APP_ENV=prod
  ```
* Or call programs with absolute paths.

### 4) SSH vs local sessions

* SSH establishes a *login* environment; terminal emulators often start *non-login* shells.
* That changes which startup files are read → different env.

### 5) Containers (Docker/Podman/K8s)

* Container processes start with env defined by the runtime:

  * `docker run -e KEY=VAL ...`
  * Kubernetes `env:` and `envFrom:` in Pod specs.
* Your host shell’s env does not magically flow inside unless you pass it.

---

## **Mini toolbelt**

### Print the environment that a *new* process would see

```bash
env | sort
```

### Compare parent vs child quickly

```bash
# Parent snapshot
env | sort > /tmp/parent.env

# Spawn a subshell and take its snapshot
bash -lc 'env | sort' > /tmp/child.env

diff -u /tmp/parent.env /tmp/child.env
```

### Inspect an already-running service (systemd)

```bash
# Get PID
systemctl show myapp.service -p MainPID

# Then:
tr '\0' '\n' < /proc/<PID>/environ | sort
```

### Quick PID by name, then print env (beware multiple matches)

```bash
pidof myapp | xargs -n1 -I{} sh -c 'echo PID={}; tr "\0" "\n" < /proc/{}/environ | sort'
```

---

## **Programmatic handoff (shell snippet)**

Wrap a command with an explicit env file, making handoff obvious:

```bash
# envfile format: KEY=VAL per line (no quotes)
run-with-env() {
  local envfile="$1"; shift
  # Export each non-empty, non-comment line
  set -a
  . "$envfile"
  set +a
  exec "$@"
}

# Usage:
# run-with-env /etc/myapp/env ./myapp --flag
```

This mirrors how `EnvironmentFile=` behaves for services.

---

## **Security notes**

* `/proc/<pid>/environ` exposes secrets to users who can read the process; keep permissions tight.
* Logs and crash dumps may capture env; don’t store sensitive tokens there unless you accept the risk.
* Some runtimes scrub env (`LD_*`, etc.) for safety; don’t rely on these for secrecy.

---

## **Mental checklist when debugging env issues**

1. **Where is the process launched from?** (shell, systemd, cron, container)
2. **Which files were read for that session type?** (`.bashrc`, `.profile`, `/etc/environment`, none?)
3. **What does `/proc/<pid>/environ` say?** Believe the kernel over your assumptions.
4. **Is `sudo` or SSH altering the env?**
5. **Is the path absolute and is `PATH` sane?**
6. **Can you reproduce with `env -i` to eliminate noise?**

