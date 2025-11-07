---
title: Content 
date: 2025-11-06
tags: 
     - linux
     - core
     - env
     - cheatsheet 
summary: An index and overview of environment variables in Linux, explaining their purpose, usage, and how they influence process behavior.
aliases:
    - Linux Environment Variables - Content Cheatsheet
---

# **Environment Variables — Index**

Environment variables are simple key–value strings handed from one process to another. Their power comes from how Linux propagates them: every process inherits its parent’s environment unless changed. This small mechanism shapes how applications behave, from shells to servers to containers.

## **What an environment variable is**

A process contains a tiny dictionary of `KEY=value` strings called its *environment*.
If a parent process sets `FOO=bar` and exports it, any child process will start life with the same `FOO=bar`.

Env vars are not files, not system settings, not permanent.
They exist only as long as the process exists.

## **Why they matter**

Env vars steer behavior without touching code:

* configure applications (`APP_MODE=prod`)
* point to paths (`JAVA_HOME`, `PATH`)
* toggle features (`DEBUG=true`)
* customize localization (`LANG`, `LC_ALL`)
* pass secret-like values (risky but common)

Frameworks like Spring treat OS env vars as a first-class configuration source, so understanding how Linux handles them becomes essential.

## **Core ideas**

1. **Exporting** makes a variable visible to child processes.
   `export VAR=value`

2. **Shell variables are not environment variables** until exported.

3. **Inheritance** is the entire game:
   parent → child → grandchild — each gets a copy.

4. **Changing your shell does not change running services.**
   Systemd services, cron jobs, containers — each has its own environment.

5. **Everything boils down to “process receives KEY=value at launch.”**
   There is no global environment store.

## **Sources of environment variables**

They normally come from:

* interactive shells and your manual `export`
* shell startup files (`~/.bashrc`, `~/.profile`)
* system-wide files (`/etc/environment`)
* `VAR=value command` one-offs
* systemd unit definitions (`Environment=`, `EnvironmentFile=`)
* container runtimes (Docker/Podman `-e`)
* login managers (graphical sessions)
* scripts that set variables before running commands

All of these ultimately feed processes the same thing: `KEY=value`.

## **What this section contains**

Each theme is covered in its own file so you can jump straight to the scenario you need.

* **setting-env.md** — defining, exporting, overriding, printing
* **persisting-env.md** — saving env vars for future shells and sessions
* **process-env.md** — inheritance mechanics, inspecting running envs
* **systemd-env.md** — env vars for services and daemons
* **env-vs-shell.md** — shell variables vs environment variables
* **security.md** — risks of secrets in env vars; leaks & mitigations
* **naming-conventions.md** — uppercase snake case, safe characters, idioms
* **env-vs-config-files.md** — when to store config in envs vs dedicated files

## **Directory layout**

Your vault structure stays consistent:

```
cheatsheets/
  linux/
    core/
      env/
        index.md                 ← this file
        setting-env.md
        persisting-env.md
        process-env.md
        systemd-env.md
        env-vs-shell.md
        security.md
        naming-conventions.md
        env-vs-config-files.md
```

## **How to use these notes**

Start with `setting-env.md` for live, practical usage.
Move to `persisting-env.md` when you want env vars to survive shell restarts.
Jump into `systemd-env.md` when your application won’t “see” env values you expect.

