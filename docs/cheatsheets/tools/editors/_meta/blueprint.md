---
title: Editor Future Blueprint
date: 2025-10-17
summary: A standardized structure and templates for documenting editor cheatsheets (IDEA, VS Code, Vim, etc.).
---



# Editor/ Blueprint

## Folder layout (future-proof)

```
cheatsheets/
└─ tools/
   └─ editors/
      ├─ idea/
      │  ├─ code-snippets.md          # Live Templates, postfix tricks, surround templates
      │  ├─ shortcuts.md              # Keymap & combos
      │  ├─ debugging.md              # Breakpoints, watches, eval expr
      │  ├─ configuration.md          # Code style, inspections, file templates
      │  ├─ run-configurations.md     # App/Test/Spring Boot configs
      │  ├─ settings-sync.md          # Sync/export settings strategy
      │  └─ refactoring.md            # Structural search/replace, safe delete, intentions
      ├─ vscode/
      │  ├─ code-snippets.md          # JSON snippets, multi-cursor patterns
      │  ├─ shortcuts.md              # Keybindings & chords
      │  ├─ debugging.md              # launch.json, DAP tips
      │  ├─ configuration.md          # settings.json, workspace vs user
      │  ├─ extensions.md             # Curated list w/ why + config
      │  └─ settings-sync.md
      └─ vim/
         ├─ code-snippets.md          # UltiSnips/Luasnip (optional)
         ├─ shortcuts.md
         └─ configuration.md
```

## File naming rules (simple + consistent)

* Keep **one concept per file**: `code-snippets.md`, `shortcuts.md`, `debugging.md`, `configuration.md`.
* Use **editor folder** to carry specificity. Inside `idea/`, `code-snippets.md` = “IntelliJ Live Templates/snippets”. Inside `vscode/`, same name = VS Code snippets.
* If a concept grows big, split with suffixes:
  `configuration-formatting.md`, `configuration-inspections.md`.

## Front matter template (uniform metadata)

Use the same front matter across all editor docs for fast filtering/search:

```md
---
title: IDEA — Code Snippets
tags: [editors, idea, code, snippets]
summary: Live Templates, postfix, and surround examples for IntelliJ IDEA.
aliases:
---
```

Change only the **title**, **tags**, and **summary** per file:

* `tags` suggestions:

  * IDEA: `[editors, idea, snippets]`, `[editors, idea, debugging]`, `[editors, idea, config]`
  * VS Code: `[editors, vscode, snippets]`, `[editors, vscode, extensions]`

## Document skeletons (copy/paste starters)

### `code-snippets.md`

```
# Code Snippets

## Scope & intent
What this page covers and how to trigger snippets.

## High-leverage snippets
- Abbrev → expansion → where it’s valid (context)
- JSON/Template blocks

## How to add/update
- Steps to create/modify
- Where files live (path)
- Sync/export strategy

## Gotchas
- Context not enabled
- Expansion key not Tab
```

### `shortcuts.md`

```
# Shortcuts

## Essentials (daily driver)
- Navigation, search, refactor

## Editing power moves
- Multi-cursor, duplicate, move line

## Debugging shortcuts
- Start/stop, step, eval

## Custom bindings you rely on
```

### `configuration.md`

```
# Configuration

## Baseline
- Code style, formatter, inspections

## Project templates / file templates
- Snippets, variables/macros

## Export/Sync
- Settings Sync, what to exclude/include
```

### `debugging.md`

```
# Debugging

## Setup
- Config types, env vars

## Techniques
- Conditional breakpoints, logpoints, watches

## Patterns
- Troubleshooting common failures
```

## Quick rename guidance (today → future)

* If you already created `live-templates.md` for IDEA, **rename to**:

  * `cheatsheets/tools/editors/idea/code-snippets.md`
* When you add VS Code later, create:

  * `cheatsheets/tools/editors/vscode/code-snippets.md`

## Optional: a tiny README index per editor

Add a minimal `README.md` inside each editor folder:

```md
---
title: IDEA — Index
tags: [editors, idea]
summary: Entry points for all IntelliJ notes.
---

- [Code Snippets](./code-snippets.md)
- [Shortcuts](./shortcuts.md)
- [Debugging](./debugging.md)
- [Configuration](./configuration.md)
- [Run Configurations](./run-configurations.md)
- [Settings Sync](./settings-sync.md)
- [Refactoring](./refactoring.md)
```

This gives you a stable, parallel structure for **IDEA**, **VS Code**, and any future editor without bikeshedding filenames. 
