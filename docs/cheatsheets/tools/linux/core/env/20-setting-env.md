---
title: Setting 
date: 2025-11-07
tags: 
    - linux
    - environment-variables
    - shell
    - bash
    - cheatsheet
summary: 
    A quick reference on how to set, export, override, and inspect environment variables in a Linux shell session.
aliases:
    - Linux Environment Variables - Setting Environment Variables Cheatsheet
---



# **Setting Environment Variables**

Environment variables are just `KEY=value` pairs attached to processes. This note covers how to define them, export them, override them, and inspect them in a running shell session. Nothing here persists after you close the terminal — persistence is in `persisting-env.md`.

## **Defining a variable (shell-only)**

Writing a variable assigns it *inside the shell*, but not to the environment:

```
FOO=hello
```

This lives only as a **shell variable**, invisible to child processes.

```
echo $FOO          # prints "hello"
printenv FOO       # prints nothing — not in the environment
```

A shell variable is like a local note on your desk: useful to you, invisible to your children.

## **Exporting a variable (becomes inherited)**

Exporting marks a variable as part of the environment:

```
export FOO=hello
```

or split style:

```
FOO=hello
export FOO
```

Now every child process will receive `FOO=hello`.

You can check:

```
printenv FOO
```

Exporting is what turns a shell-local value into something visible to everything you launch from this shell.

## **Modifying or overriding a variable**

Overriding uses the same syntax:

```
export FOO=changed
```

Child processes will now see the new value.

Your current shell also changes:

```
echo $FOO          # changed
```

If you accidentally set a shell variable without export:

```
FOO=local
echo $FOO          # local
printenv FOO       # still "changed" (the old env value)
```

Shell variable shadows the environment variable inside the shell session, but children still inherit the exported version unless you re-export.

## **Setting variables for a single command**

You can attach variables to one command without polluting the environment:

```
FOO=hello BAR=world command
```

Example:

```
PORT=9000 APP_MODE=dev ./myapp
```

After the command finishes, the variables vanish.
This is the cleanest way to test an application with different env values.

## **Unsetting variables**

To remove a variable from the environment:

```
unset FOO
```

This deletes both shell and environment versions in your current session.

You can confirm:

```
printenv FOO
echo $FOO
```

Both empty.

## **Inspecting the environment**

There are multiple ways, each for a slightly different viewpoint:

### Global view of environment variables

```
printenv
```

### Or filter one variable

```
printenv PATH
printenv JAVA_HOME
```

### View shell + environment variables (a superset)

```
set
```

### Launch a command with a modified environment and inspect inside

```
env FOO=123 bash
```

Inside that subshell:

```
echo $FOO
```

The parent shell won’t see it.

## **Launching programs with a clean environment**

Sometimes you want to wipe the environment and start fresh:

```
env -i bash
```

This launches a bash shell with no environment variables except what bash itself injects.

Useful for debugging weird `$PATH` issues.

## **Subshells and inheritance**

A subshell inherits the environment but not changes you make after it starts.

Example:

```
export FOO=1
bash             # start subshell
```

Inside subshell:

```
echo $FOO        # 1
```

If in the *parent* shell you change:

```
export FOO=2
```

The subshell still sees `1` — it inherited at launch and never updates.

Inheritance happens once per process at startup.

## **Good practice patterns**

* Use UPPERCASE for env variables.
* Export only what you intend children to see.
* Use `VAR=value command` for temporary test runs.
* Use `printenv` to verify what a child will receive.
* Keep `PATH` changes deliberate: exporting malformed PATH breaks many tools.

