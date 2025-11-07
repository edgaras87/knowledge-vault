---
title: Env vs Shell
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - shell
    - bash
    - cheatsheet
summary: A quick reference on the difference between environment variables and shell variables in Linux, including how they interact, inheritance rules, and common pitfalls.
aliases:
    - Linux Environment Variables - Environment Variables vs Shell Variables Cheatsheet
---

# **Environment Variables vs Shell Variables**

The shell has its own little universe of variables. The environment is a different universe. They overlap, but only when you *export* does a shell variable step into the environment.

This page explains the difference, the inheritance rules, and the weird behaviors you meet in real Linux systems.

---

## **Two worlds**

### **Shell variables**

* Exist *only inside the running shell process*.
* Created simply by writing:

  ```
  FOO=hello
  ```
  
* Not inherited by children unless exported.
* Stored in the shell’s own data structures (Bash’s symbol table).
* Disappear when the shell exits.

### **Environment variables**

* Stored in the process’s environment block.
* Inherited by child processes.
* Created only when you `export`:

  ```
  export FOO=hello
  ```
  
* Visible to any program launched from the shell (`printenv`, apps, scripts).

**Key idea:**

A shell variable is local memory; an environment variable is part of the process contract.

---

## **What `export` actually does**

`export` marks a variable to be **copied into the environment** during the next process launch:

```
FOO=hello     # shell variable only
export FOO    # now FOO will appear in env for child processes
```

This transforms:

```
FOO=hello  →  FOO=hello (environment)
```

The original shell variable survives; the export just mirrors it into the environment block.

---

## **Inspecting the difference**

### Shell variable:

```
FOO=abc
echo $FOO      # prints abc
printenv FOO   # empty — not exported
```

### Environment variable:

```
export FOO=abc
printenv FOO   # abc
```

### Seeing both:

```
set | grep FOO
printenv | grep FOO
```

`set` → shell + exported env
`printenv` → only environment

---

## **Shadowing: when shell hides environment**

If you do:

```
export FOO=prod
FOO=dev
```

Then:

```
echo $FOO      # dev  (shell variable shadows)
printenv FOO   # prod (environment still holds old value)
```

To update both:

```
export FOO=dev
```

---

## **Command-local environment**

```
FOO=123 BAR=abc command
```

This **does not** create shell variables.

It builds a temporary environment only for `command`:

* After `command`, the vars disappear.
* Your shell is unaffected.
* Children of `command` inherit the temporary env.

Useful for testing apps without polluting your session.

---

## **Subshells and inheritance**

Starting a subshell:

```
bash
```

It inherits your environment but **not your shell variables**.

Example:

```
FOO=local
export BAR=env
bash
```

Inside subshell:

```
echo $FOO     # empty
echo $BAR     # env
```

Shell variables do not cross process boundaries.

Environment variables do.

This is the foundation of process-level configuration.

---

## **Why systemd ignores your shell**

Your shell evaluates `~/.bashrc`, `~/.profile`, PATH modifications, aliases, functions, exports…
Systemd does none of this. It launches services directly, with its own environment, isolated from your shell worlds.

**New process → new chain of inheritance.**

Your terminal has nothing to do with system services unless you explicitly define env in the unit.

---

## **Why scripts behave differently when run manually**

When you run:

```
./run.sh
```

The script inherits your environment.

But when cron or systemd runs it:

* PATH is different
* TERM may not exist
* LANG may be empty
* Your exported variables are missing

Different parent → different inheritance → different behavior.

---

## **Shell expansions ≠ environment variables**

Shell variables can do expansions like:

```
FOO="a $HOME b"
```

Environment variables don’t expand themselves; they are **literal** strings passed to child processes.

The expansion happens in the shell **before** the value enters the environment.

---

## **Practical patterns**

### Assign shell-only values:

```
TMP=foo
```

### Assign and export (environment):

```
export TMP=foo
```

### Temporary env for a single command:

```
TMP=foo OTHER=bar ./app
```

### Debug what a script *actually* sees:

```
env | sort > /tmp/pre.env
bash -c 'env | sort > /tmp/in.env'
diff -u /tmp/pre.env /tmp/in.env
```

### Confirm service environment:

```
pid=$(systemctl show myapp.service -p MainPID --value)
tr '\0' '\n' < /proc/$pid/environ | sort
```

---

## **Mental model to keep forever**

- The shell has variables.
- The environment has variables.
- Only exported variables move from shell → child processes.
- System services, cron, containers create *their own* parent environment.
- Inheritance happens once at process start; never updates afterward.

