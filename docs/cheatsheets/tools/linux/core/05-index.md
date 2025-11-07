---
title: Content
date: 2025-11-06
tags: 
    - linux
    - core
    - cheatsheet
summary: A cheat sheet for core Linux concepts, detailing fundamental mechanisms like processes, environment variables, signals, permissions, and shells that underpin the operating system's behavior.
aliases:
  - Linux Core - Index Cheatsheet
---

# **Linux Core — Index**

This section captures the ground beneath everything else in Linux: processes, environments, signals, permissions, shells, and the small invisible rules that make the system behave. These are not tools — these are the physics of the operating system. Understanding them makes everything above (Docker, Spring, servers, databases) feel simpler and less mysterious.

## **What “core” means**

Core Linux concepts are the mechanisms always running in the background:

* how processes start, stop, inherit state
* how environment variables travel
* how signals shape process behavior
* how users, groups, and permissions gate access
* how shells interpret commands
* how filesystems and paths resolve

Nothing here is optional. Every higher-level tool ultimately leans on these fundamentals.

## **Why this matters**

When you grasp the core, debugging stops being guesswork.
You understand:

* why a service doesn’t see your environment variables
* why a script behaves differently when run manually vs systemd
* why file permissions break deployments
* why `$PATH` magically changes between sessions
* why signals (SIGTERM, SIGINT) behave differently
* why your app works in a shell but fails in a container

Core knowledge is leverage. Everything becomes easier.

## **How this section is structured**

Each concept lives in its own folder. Small, targeted, scenario-based cheat-sheets:

```
env/           # Environment variables (environment model, inheritance, systemd)
processes/     # Processes, PIDs, lifecycle, /proc, forking, exec, zombies (future)
signals/       # SIGINT, SIGTERM, SIGHUP, traps, handlers (future)
permissions/   # users, groups, chmod, umask (future)
paths/         # filesystem paths, absolute vs relative, symlinks (future)
shell/         # shell mechanics, quoting, expansion, subshells (future)
```

## **Inheritance: the unifying idea**

A surprising amount of Linux behavior can be understood through one pattern:

**Parent → Child inheritance.**
Processes, environment variables, open file descriptors, ulimits, signals — many things follow this chain.
If something “works in one context but not another,” it usually means the inheritance path changed.

## **What belongs here**

Anything describing how Linux behaves *before* tools or frameworks enter the picture.

Examples of future topics you may add:

* how `/proc` exposes process internals
* how login shells differ from non-login shells
* how systemd interacts with environment variables and signals
* how `$PATH` is built
* how file permission bits translate to real access
* how subshells and pipes form isolated environments

Each gets its own cheatsheet.

## **Where this lives in your vault**


```
cheatsheets/
  linux/
    core/
      index.md      ← this file
      env/
        index.md
        ...
```

