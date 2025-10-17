---
title: VS Code Snippets
date: 2025-10-17
tags: 
    - vscode
    - snippets
    - templates
    - productivity
summary: A compact guide to using and creating code snippets and file templates in Visual Studio Code.
aliases:
  - VS Code Snippets

---

# 💡 VS Code — Code Snippets Cheatsheet

---

## 🧭 Introduction

VS Code supports two complementary systems for automation:

1. **User Snippets** — dynamic, context-aware pieces of code or text that you trigger manually inside existing files.
   They live in JSON files and expand when you type their **prefix** and hit **Tab**.

2. **File Templates (via Extensions or Snippet Files)** — prefilled structures for new files, often provided by extensions like *Project Templates* or *File Templates Generator*.
   These use the same variable system but apply automatically when you create a file.

Together, they make VS Code behave like a lightweight IDE: boilerplate vanishes, consistency remains.

---

## 🧩 PART 1 — User Snippets

---

### ⚙️ What User Snippets Are

User snippets are stored in JSON files per language (or globally).
Each snippet includes a **prefix**, **body**, and **description**.
Variables like `TM_FILENAME` or `CURRENT_YEAR` can be embedded directly in the body.

**Path:**
`File → Preferences → User Snippets` → choose language or `New Global Snippets file…`

---

### 🧱 Example — Markdown Front Matter with Auto Date

**File:** `markdown.json`

```json
{
  "YAML Front Matter": {
    "prefix": "yfm",
    "description": "YAML front matter for Markdown",
    "body": [
      "---",
      "title: ${1:title}",
      "date: ${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE}",
      "tags:",
      "summary: ${2:summary}",
      "aliases:",
      "---",
      "$0"
    ]
  }
}
```

**Usage:**
Type `yfm` → Tab → VS Code expands to:

```yaml
---
title: My Note
date: 2025-10-17
tags:
summary: Short intro
aliases:
---
```

---

### ⚡ Triggering Snippets

| Action                  | Shortcut                                       | Description                               |
| ----------------------- | ---------------------------------------------- | ----------------------------------------- |
| Expand snippet          | `Tab`                                          | Type prefix and press Tab                 |
| Show available snippets | `Ctrl + Space` (Win/Linux) / `⌘ Space` (macOS) | Lists snippets valid for the current file |
| Edit snippets           | `Ctrl + Shift + P → Configure User Snippets`   | Opens snippet files                       |

---

### 🧰 Common VS Code Snippet Variables

| Variable               | Example Output           | Description                       |
| ---------------------- | ------------------------ | --------------------------------- |
| `$CURRENT_YEAR`        | `2025`                   | Current year                      |
| `$CURRENT_MONTH`       | `10`                     | Current month                     |
| `$CURRENT_DATE`        | `17`                     | Current day                       |
| `$CURRENT_HOUR`        | `14`                     | Current hour                      |
| `$CURRENT_MINUTE`      | `32`                     | Current minute                    |
| `$CURRENT_SECOND`      | `07`                     | Current second                    |
| `$TM_FILENAME`         | `UserService.java`       | File name                         |
| `$TM_FILENAME_BASE`    | `UserService`            | File name without extension       |
| `$TM_LINE_NUMBER`      | `42`                     | Current line number               |
| `$TM_SELECTED_TEXT`    | *(selected code)*        | Selected text                     |
| `$TM_CURRENT_WORD`     | *(word under cursor)*    | Current word                      |
| `$CLIPBOARD`           | *(clipboard text)*       | Clipboard contents                |
| `$RANDOM`              | random integer           | Random value                      |
| `$UUID`                | `2a4e...`                | Unique identifier                 |
| `$BLOCK_COMMENT_START` | `/*`                     | Language’s block comment start    |
| `$BLOCK_COMMENT_END`   | `*/`                     | Language’s block comment end      |
| `$TM_DIRECTORY`        | `/home/user/project/src` | Directory of the current file     |
| `$RELATIVE_FILEPATH`   | `src/app.js`             | Relative path from workspace root |
| `$WORKSPACE_NAME`      | `my-project`             | Workspace folder name             |
| `${1:placeholder}`     | `cursor tabstop`         | Field for manual input            |
| `$0`                   | —                        | Final cursor position             |

**Pro tip:**
You can nest variables:
`"${TM_FILENAME_BASE}_${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE}"` →
`UserService_2025-10-17`

---

### 💡 Power Tips for Snippets

* Add tab stops `${1}`, `${2}` … and a final `$0` to control cursor flow.
* Use `${CLIPBOARD}` to instantly insert what’s copied.
* For optional defaults: `${1:defaultValue}` → replaced when you type.
* Multi-line snippets use an array of strings — each line quoted.
* Use language-specific snippet files (`markdown.json`, `java.json`, etc.) to scope snippets.

---

## 🏗️ PART 2 — File Templates (Projects and New Files)

---

### ⚙️ What File Templates Are

VS Code doesn’t have a native “File Template” system like IntelliJ,
but several extensions provide similar behavior:

* **Project Templates** by zardoy — create project/file blueprints.
* **Snippet Templates** or **Advanced New File** — insert predefined content on file creation.
* Or simply use a global snippet with the command **Insert Snippet** (`Ctrl + Shift + P`).

All use the same variables as normal snippets.

---

### 🧱 Example — Markdown Note Template

Using the same YAML header pattern for new Markdown files:

```json
{
  "New MD Note": {
    "prefix": "newnote",
    "description": "Prefilled Markdown note structure",
    "body": [
      "---",
      "title: ${TM_FILENAME_BASE}",
      "date: ${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE} ${CURRENT_HOUR}:${CURRENT_MINUTE}",
      "tags:",
      "summary:",
      "aliases:",
      "---",
      "",
      "$0"
    ]
  }
}
```

**Usage:**
Type `newnote` in a new empty file → Tab → prefilled note.

---

### 📘 Common Patterns for Templates

* **Timestamps:** `${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE}_${CURRENT_HOUR}${CURRENT_MINUTE}`
* **User header:** `Author: ${USER}` (works if environment variable set)
* **Filename metadata:** `File: ${TM_FILENAME}`
* **Unique ID:** `id: ${UUID}`

---

## 🧭 Quick Reference

| System                            | Purpose                             | Syntax              | Trigger                    |
| --------------------------------- | ----------------------------------- | ------------------- | -------------------------- |
| **User Snippet**                  | Expand snippet inside existing file | `${VARIABLE}`       | Prefix + Tab               |
| **File Template (via extension)** | Prefilled new file content          | `${VARIABLE}`       | New File command / snippet |
| **Multi-cursor / Selection**      | Insert across many lines            | `$TM_SELECTED_TEXT` | Works with multi-cursor    |
| **Clipboard Insert**              | Paste clipboard text                | `$CLIPBOARD`        | Inside snippet body        |

---

## 🧠 Summary

**User Snippets** → dynamic, context-aware expansions inside existing files.
Use VS Code variables like `$CURRENT_YEAR`, `$TM_FILENAME_BASE`, `$UUID`, and `$CLIPBOARD` to insert live data.

**File Templates** → prefilled structures for new files, implemented via extensions or global snippets.
Use the same `${VARIABLE}` syntax for consistency.

Both systems integrate seamlessly — create once, reuse forever.
VS Code’s JSON-based approach makes snippets portable, version-controlled, and easy to share across machines.


