---
title: IDEA Code Snippets
tags: 
    - editors
    - idea
    - snippets
    - templates
summary: How to create and use Live Templates and File Templates in IntelliJ IDEA.
aliases:
    - IntelliJ IDEA Code Snippets
---

# 💡 IntelliJ IDEA — Code Snippets (Live Templates & File Templates)

---

## 🧭 Introduction

IntelliJ IDEA supports two powerful snippet systems:

1. **Live Templates** — dynamic, context-aware snippets you trigger manually inside existing files (e.g., typing `yfm` → Tab). They use IntelliJ **macros** like `date()`, `className()`, and `clipboard()` to inject live data.

2. **File Templates** — static, prefilled structures used when creating new files. They use **Velocity-style variables** like `${NAME}`, `${DATE}`, and `${YEAR}-${MONTH}-${DAY}`.

Together, they automate everything from boilerplate code to note headers and logging patterns — turning IntelliJ into a serious productivity engine.

---

## 🧩 PART 1 — Live Templates

---

### ⚙️ What Live Templates Are

**Live Templates** let you expand small abbreviations into complete code or text blocks.
They’re ideal for repeating structures (YAML headers, annotations, logging, test stubs, etc.) and can include dynamic macros that automatically fill in data such as the date, file name, or user.

**Path:**
`Settings → Editor → Live Templates`

---

### 🧱 Example — Markdown Front Matter with Auto Date

**Abbreviation:** `yfm`

**Template text:**

```
---
title: $TITLE$
date: $DATE$
tags: 

summary: $SUMMARY$
aliases:

---
$END$
```

**Variable setup (Edit Variables…):**

| Variable  | Expression           | Stop at? | Description             |
| --------- | -------------------- | -------- | ----------------------- |
| `TITLE`   | *(empty)*            | ✅        | You fill it manually    |
| `DATE`    | `date("yyyy-MM-dd")` | ❌        | Auto-fills today’s date |
| `SUMMARY` | *(empty)*            | ✅        | Optional summary        |

Now type `yfm` → press **Tab** → IntelliJ expands to:

```
---
title: 
date: 2025-10-17
tags: 

summary: 
aliases:

---
```

---

### ⚡ Insert or Trigger a Live Template

| Action                   | Shortcut                              | Description                               |
| ------------------------ | ------------------------------------- | ----------------------------------------- |
| Expand template          | `Tab`                                 | Type abbreviation and press Tab           |
| Show available templates | `Ctrl + J` (Win/Linux) / `⌘J` (macOS) | Lists all templates valid in this context |
| Surround selected text   | `Ctrl + Alt + J` / `⌥⌘J`              | For templates with `$SELECTION$`          |
| Manage templates         | `Settings → Editor → Live Templates`  | Template editor                           |

---

### 🧰 Commonly Used IntelliJ Macros

| Macro                        | Example Output     | Description                     |
| ---------------------------- | ------------------ | ------------------------------- |
| `date("yyyy-MM-dd")`         | `2025-10-17`       | Current date (custom format)    |
| `time("HH:mm")`              | `08:42`            | Current time                    |
| `user()`                     | `edgaras`          | Your system/IDE username        |
| `clipboard()`                | *(clipboard text)* | Pastes clipboard content        |
| `className()`                | `MainController`   | Current class name              |
| `methodName()`               | `getUserById`      | Current method name             |
| `packageName()`              | `com.example.app`  | Current package                 |
| `fileName()`                 | `UserService.java` | File name                       |
| `fileNameWithoutExtension()` | `UserService`      | File name stripped of extension |
| `uuid()`                     | `2a4e...`          | Generates a UUID                |
| `selection()`                | *(selected code)*  | Used in Surround templates      |
| `capitalize(…)`              | `Hello`            | Capitalizes text                |
| `snakeCase(…)`               | `my_variable`      | Converts to snake_case          |
| `camelCase(…)`               | `myVariable`       | Converts to camelCase           |
| `prompt("Label")`            | *(asks user)*      | Prompts for input               |

---

### 💡 Power Tips

* Prefix abbreviations by category (`md_`, `j_`, `r_`, etc.) to keep lists organized.
* `$END$` marks where the cursor lands after expansion.
* `$SELECTION$` allows templates that wrap selected text.
* Combine macros:

  * `capitalize(fileNameWithoutExtension())` → `MyFile`
  * `camelCase(clipboard())` → convert copied text to variable name
  * `uuid().substring(0,8)` → short random ID

---

## 🏗️ PART 2 — File Templates

---

### ⚙️ What File Templates Are

**File Templates** define prefilled content for *new* files.
When you create a new file (e.g., “New → MD Note”), IntelliJ uses these templates to populate default text.

They use a simpler syntax — **Velocity variables** — which look like `${VARIABLE}`.
These are resolved at creation time, not live while editing.

**Path:**
`Settings → Editor → File and Code Templates`

---

### 🧱 Example — Markdown Note Template

**Name:** `MD Note`
**Extension:** `md`

**Template text:**

```
---
title: ${NAME}
date: ${YEAR}-${MONTH}-${DAY}
tags: 
summary: 
aliases:
---
```

**Result when creating a new file:**

```
---
title: my-new-note
date: 2025-10-17
tags: 
summary: 
aliases:
---
```

---

### 📘 Common File Template Variables

| Variable          | Example Output | Description            |
| ----------------- | -------------- | ---------------------- |
| `${NAME}`         | `my-file`      | New file name          |
| `${USER}`         | `edgaras`      | Current system user    |
| `${DATE}`         | `17/10/2025`   | Localized date         |
| `${TIME}`         | `09:12`        | Current time           |
| `${YEAR}`         | `2025`         | Current year           |
| `${MONTH}`        | `10`           | Current month          |
| `${DAY}`          | `17`           | Current day            |
| `${PACKAGE_NAME}` | `com.example`  | Java package           |
| `${CLASS_NAME}`   | `UserService`  | Derived from file name |

**Velocity Tip:** You can use expressions like:

```
${YEAR}-${MONTH}-${DAY}_${TIME}
```

to generate unique timestamps for file names.

---

### 💡 Power Tips

* Use File Templates for your recurring file types: configuration files, test classes, documentation stubs.
* Use `${DATE}` for locale-aware date or `${YEAR}-${MONTH}-${DAY}` for ISO-style.
* Add **includes** for shared blocks (Settings → File and Code Templates → Includes tab).
* Works great combined with **Live Templates** — start from a File Template, enhance later with dynamic snippets.

---

## 🧭 Quick Reference

| Template Type         | Purpose                                      | Syntax        | Trigger                |
| --------------------- | -------------------------------------------- | ------------- | ---------------------- |
| **Live Template**     | Dynamic snippets inside existing files       | `$VARIABLE$`  | Abbrev + Tab           |
| **File Template**     | Prefilled structure for new files            | `${VARIABLE}` | File → New             |
| **Postfix Template**  | Inline transformation (`.if`, `.for`, `.nn`) | —             | After expression + Tab |
| **Surround Template** | Wrap selected code                           | `$SELECTION$` | `Ctrl+Alt+J` / `⌥⌘J`   |

---

## 🧠 Summary

**Live Templates** — dynamic, context-aware snippets for existing files.
Use IntelliJ **macros** such as `date()`, `className()`, `clipboard()`, `uuid()`, and `user()` to auto-fill information.

**File Templates** — static, prefilled structures for new files.
Use **Velocity-style variables** like `${NAME}`, `${DATE}`, `${YEAR}-${MONTH}-${DAY}` to scaffold default content.

Both systems complement each other:

* *Live Templates* automate repetitive typing.
* *File Templates* provide consistent starting points.
  Together, they’re the backbone of a fast, error-free workflow inside IntelliJ IDEA.

