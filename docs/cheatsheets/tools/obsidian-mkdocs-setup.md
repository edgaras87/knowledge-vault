---
title: obsidian-mkdocs-setup
tags: [cheatsheet, tools, obsidian, mkdocs, documentation]
summary: Setup guide for using Obsidian with MkDocs Material to create a developer docs site.
---


# Obsidian ↔ MkDocs Material — Setup & Structure (for `knowledge-vault/docs`)

**Goal:** Write in Obsidian with `[[wikilinks]]` → publish a polished docs site via MkDocs Material.

**Two modes:** **Cheatsheets** (quick lookup) and **Concepts** (deep understanding). Keep them separate but cross-linked.

---

## 0) Prereqs

* Python 3.8+
* Git
* Obsidian app (optional but recommended for writing)

---

## 1) Repo layout (the shape we’re aiming for)

```
knowledge-vault/
├─ mkdocs.yml                # site config (we’ll create it below)
├─ README.md                 # for GitHub visitors (how to run/deploy)
└─ docs/                     # all site content lives here
   ├─ index.md               # site homepage
   ├─ cheatsheets/           # quick references (APIs, commands, patterns)
   │  ├─ index.md            # cheatsheets landing page
   │  ├─ languages/
   │  │  ├─ java/
   │  │  │  ├─ core/
   │  │  │  │  └─ streams.md
   │  │  │  └─ frameworks/
   │  │  │     └─ spring/
   │  │  │        ├─ annotations.md
   │  │  │        └─ rest-controller.md
   │  │  └─ python/
   │  │     └─ basics.md
   │  ├─ databases/
   │  │  ├─ sql/
   │  │  │  ├─ basics.md
   │  │  │  └─ joins.md
   │  │  ├─ mysql/
   │  │  │  ├─ setup/
   │  │  │  │  ├─ java.md
   │  │  │  │  └─ python.md
   │  │  │  └─ queries.md
   │  │  └─ orm/
   │  │     ├─ jpa-java-annotations.md
   │  │     └─ sqlalchemy-python-cheats.md
   │  ├─ networking/
   │  │  └─ http/
   │  │     ├─ basics.md
   │  │     └─ headers.md
   │  └─ tools/
   │     ├─ git.md
   │     ├─ docker.md
   │     └─ obsidian-mkdocs-setup.md    ← this cheatsheet
   └─ concepts/              # deeper explanations and trade-offs
      ├─ index.md            # concepts landing page
      ├─ backend/
      │  ├─ http.md
      │  ├─ rest-api.md
      │  └─ caching.md
      ├─ databases/
      │  ├─ normalization.md
      │  ├─ indexing.md
      │  ├─ transactions-acid.md
      │  └─ orm/
      │     ├─ orm-concepts.md
      │     └─ jpa-vs-sqlalchemy.md
      ├─ frameworks/
      │  ├─ spring-core.md
      │  └─ hibernate.md
      └─ design/
         ├─ dependency-injection.md
         └─ microservices.md
```

**Why this structure?** Cheatsheets = fast lookup (language/framework/API); Concepts = how/why/architecture. It mirrors how your brain flips between coding and understanding, and keeps search results clean.

---

## 2) One-time install

```bash
cd knowledge-vault

# (Optional) keep Python deps isolated
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

python -m pip install --upgrade pip

# Core: MkDocs + Material + "Last updated" git plugin
pip install -U mkdocs mkdocs-material mkdocs-git-revision-date-localized-plugin

# Pick ONE wikilinks plugin (to convert [[Page]] to proper links):

# Option A (popular): Roam/Obsidian-style wikilinks + Obsidian-style callouts
pip install mkdocs-roamlinks-plugin mkdocs-callouts

# Option B (also good): Wikilinks via "ezlinks" (plugin name is mkdocs-wikilinks-plugin)
# pip install mkdocs-wikilinks-plugin
```

Open the repo as an **Obsidian vault** (Obsidian → Open folder as vault → select `knowledge-vault/`).
Recommended Obsidian settings:

* Files & Links → **Use [[Wikilinks]]**: On
* Files & Links → **New link format**: Shortest
* Files & Links → **Default location for new notes**: Same folder as current file
* Editor → **Show frontmatter**: On

---

### 2.5) Obsidian setup (optional but useful)

**Core plugins to enable**

* Backlinks, Outgoing Links → fast x-ref navigation
* Templates → front-matter snippets
* Daily Notes (optional) → scratchpad/logs that won’t be published

**Community plugins that play nicely with MkDocs Material**

* **Admonition** — renders MkDocs `!!! note|tip|warning` blocks *inside Obsidian*.
  Use this in your notes so one syntax works everywhere:

  ```md
  !!! tip
      Use the search bar for method names, error snippets, or concepts.
  ```

* **Advanced Tables** *(or Table Editor 2)* — auto-align pipes, tab to next column.

* **Linter** — auto-fix headings, trailing spaces, YAML ordering (keep `title/tags/summary` neat).

* **Templater** — quick scaffolds for cheatsheets/concepts.

  * Example templates:

    ```md
    ---
    title: <% tp.file.title %> — Quickref
    tags: [cheatsheet, <topic>, <tech>]
    summary: One-liner for why this exists.
    ---
    # <% tp.file.title %> — Quickref

    > See concepts: [[concepts/...]]
    ```

    ```md
    ---
    title: <% tp.file.title %>
    tags: [concept, <domain>]
    summary: What it is, why it matters, trade-offs.
    ---
    # <% tp.file.title %>

    > See cheatsheet: [[cheatsheets/...]]
    ```

* **Markdown Attributes** — previews `{#id .class}` so Obsidian shows what `attr_list` does in MkDocs.

* **Paste URL into Selection** — speeds up linking text → `[text](url)`.

* **Tag Wrangler** — bulk-rename/merge tags.

* **Obsidian Git** *(optional)* — commit/pull from inside Obsidian if you’re not using IDEA for VCS.

**Nice-to-have (safe, but non-exporting)**

* **Dataview** — dashboards/lists in the vault. Remember: Dataview queries **don’t render in MkDocs**. Use it for in-vault discovery, not for pages you publish, or export the results as static lists before publishing.

**Syntax alignment cheats**

* **Admonitions:** Prefer MkDocs syntax (`!!! note`) + Admonition plugin → same source renders well in both places. Avoid Obsidian callouts (`> [!NOTE]`) if you want one true syntax.
* **Tabs:** `pymdownx.tabbed` looks great on the site but won’t render as tabs inside Obsidian. In notes, keep tab blocks short; Obsidian will show them as plain headings—good enough for editing.
* **Wikilinks:** Keep writing `[[wikilinks]]`. `mkdocs-obsidian` resolves them on build, so reorganizing folders won’t break links.

**Ignore these in publish**
Your `.obsidian/` stays out of the site—already covered by:

```
excluded_dirs: ['.obsidian', '.trash']
```

and `.gitignore` should include:

```
.obsidian/
.trash/
```

**Gotchas**

* Dataview, buttons, or any Obsidian-only syntax isn’t rendered by MkDocs. Keep publishable pages in plain Markdown + MkDocs features.
* If Linter rewrites YAML keys, make sure it doesn’t nuke custom fields you care about.
* If you switch to Obsidian callouts, you’ll need a pre-processor to convert them to `!!!` blocks; staying with `!!!` avoids that whole dance.

---

## 3) Create folders fast (scaffold)

> Run in repo root (`knowledge-vault/`). Adjust to taste.

```bash
mkdir -p docs/{cheatsheets/{languages/{java/{core,frameworks/spring},python},databases/{sql,mysql/setup,orm},networking/http,tools},concepts/{backend,databases/orm,frameworks,design}}
```

---

## 4) `mkdocs.yml` — fully commented

Create `knowledge-vault/mkdocs.yml` with this content:

```yaml
# mkdocs.yml — Site configuration for "Knowledge Vault"
# MkDocs reads Markdown from ./docs and outputs static HTML into ./site

site_name: Knowledge Vault                 # Shown in header and metadata
site_url: https://<your-username>.github.io/knowledge-vault  # Used for canonical links/sitemaps
repo_url: https://github.com/<your-username>/knowledge-vault  # “Edit on GitHub” links
edit_uri: edit/main/docs/                  # Path to open files in GitHub’s editor
use_directory_urls: true                   # Pretty URLs: /path/ instead of /path.html

theme:
  name: material                           # Material for MkDocs theme (feature-rich)
  language: en
  features:
    - navigation.instant                   # Faster page transitions
    - navigation.tracking                  # Highlight active section as you scroll
    - navigation.sections                  # Group pages by top-level sections
    - navigation.tabs                      # Top-level sections as tabs
    - toc.integrate                        # Merge page TOC into the sidebar
    - content.code.copy                    # Copy button on code blocks
    - content.code.annotate                # Inline annotations on code
    - content.action.edit                  # “Edit this page” button
    - search.suggest                       # Search autocomplete
    - search.highlight                     # Highlight matches on page

  # Color schemes (light/dark) that follow OS preference
  palette:
    # --- Scheme 1: Light (shown when OS prefers light) ---
    - media: "(prefers-color-scheme: light)"  # follow OS light mode
      scheme: default                          # Material's default light scheme
      primary: indigo                          # header / accents
      accent: indigo                           # buttons / highlights
      toggle:
        icon: material/weather-night           # icon shown while in light mode
        name: Switch to dark mode              # accessible label (tooltip)

    # --- Scheme 2: Dark (shown when OS prefers dark) ---
    - media: "(prefers-color-scheme: dark)"   # follow OS dark mode
      scheme: slate                            # Material's dark scheme
      primary: indigo
      accent: indigo
      toggle:
        icon: material/weather-sunny           # icon shown while in dark mode
        name: Switch to light mode

# docs_dir defaults to "docs". Keeping it implicit = cleaner config.
# docs_dir: docs

plugins:
  - search                                 # Full-text search
  # If you installed Option A:
  - roamlinks
  - callouts
  # If you installed Option B instead:
  # - ezlinks
  - git-revision-date-localized:           # “Last updated” timestamps
      fallback_to_build_date: true

markdown_extensions:
  - admonition                             # !!! note/tip/warning blocks
  - attr_list                              # {#id .class} on elements
  - def_list                               # Definition lists
  - md_in_html                             # Markdown inside HTML blocks
  - tables                                 # Advanced tables
  - toc:
      permalink: true                      # Link anchors for headings
  - pymdownx.details                       # <details> collapsible sections
  - pymdownx.highlight:
      anchor_linenums: true                # Clickable line numbers in code
      line_spans: __span                   # For precise CSS targeting
  - pymdownx.inlinehilite                  # `==inline code==` highlighting
  - pymdownx.magiclink                     # Autolink URLs and issues
  - pymdownx.superfences                   # Fenced code blocks inside lists, tabs
  - pymdownx.tabbed:
      alternate_style: true                # Nice UI for tabbed code examples
  - pymdownx.tasklist:
      custom_checkbox: true                # Pretty checkboxes in lists

# Navigation:
# Option A (recommended): start minimal and let filesystem drive nav while drafting — comment out nav.
# Option B: curate the nav to control order/labels (uncomment to use).
#
# nav:
#   - Home: index.md
#   - Cheatsheets:
#       - Overview: cheatsheets/index.md
#       - Languages:
#           - Java:
#               - Streams: cheatsheets/languages/java/core/streams.md
#               - Spring Annotations: cheatsheets/languages/java/frameworks/spring/annotations.md
#               - REST Controller: cheatsheets/languages/java/frameworks/spring/rest-controller.md
#           - Python:
#               - Basics: cheatsheets/languages/python/basics.md
#       - Databases:
#           - SQL:
#               - Basics: cheatsheets/databases/sql/basics.md
#               - Joins: cheatsheets/databases/sql/joins.md
#           - MySQL:
#               - Setup (Java): cheatsheets/databases/mysql/setup/java.md
#               - Setup (Python): cheatsheets/databases/mysql/setup/python.md
#               - Queries: cheatsheets/databases/mysql/queries.md
#           - ORM:
#               - JPA Annotations (Java): cheatsheets/databases/orm/jpa-java-annotations.md
#               - SQLAlchemy Cheats (Python): cheatsheets/databases/orm/sqlalchemy-python-cheats.md
#       - Networking:
#           - HTTP Basics: cheatsheets/networking/http/basics.md
#           - Headers: cheatsheets/networking/http/headers.md
#       - Tools:
#           - Git: cheatsheets/tools/git.md
#           - Docker: cheatsheets/tools/docker.md
#           - Obsidian + MkDocs Setup: cheatsheets/tools/obsidian-mkdocs-setup.md
#   - Concepts:
#       - Overview: concepts/index.md
#       - Backend:
#           - HTTP: concepts/backend/http.md
#           - REST API: concepts/backend/rest-api.md
#           - Caching: concepts/backend/caching.md
#       - Databases:
#           - Normalization: concepts/databases/normalization.md
#           - Indexing: concepts/databases/indexing.md
#           - Transactions (ACID): concepts/databases/transactions-acid.md
#           - ORM:
#               - ORM Concepts: concepts/databases/orm/orm-concepts.md
#               - JPA vs SQLAlchemy: concepts/databases/orm/jpa-vs-sqlalchemy.md
#       - Frameworks:
#           - Spring Core: concepts/frameworks/spring-core.md
#           - Hibernate: concepts/frameworks/hibernate.md
#       - Design:
#           - Dependency Injection: concepts/design/dependency-injection.md
#           - Microservices: concepts/design/microservices.md
```

**Why comment out `nav:` at first?** While you’re building content, filesystem ordering is simpler. Later, un-comment `nav:` to curate labels and order.

---

## 5) Homepages (three `index.md` files)

> `index.md` turns a folder into a **landing page** and gives you clean URLs:
>
> * `docs/index.md` → `/`
> * `docs/cheatsheets/index.md` → `/cheatsheets/`
> * `docs/concepts/index.md` → `/concepts/`

### 5.1 `docs/index.md` (site homepage)

```markdown
---
title: Knowledge Vault
summary: Personal developer wiki — cheatsheets for speed, concepts for mastery.
---

# 🧠 Knowledge Vault

Two modes, one brain:
- **Cheatsheets** → quick reference while coding
- **Concepts** → deeper understanding and architecture

## 🚪 Start Here
- [[cheatsheets/languages/java/core/streams|Java Streams — Quickref]]
- [[cheatsheets/databases/sql/basics|SQL Basics — Cheatsheet]]
- [[concepts/backend/http|HTTP — Concepts]]
- [[cheatsheets/tools/git|Git — Commands]]
- [[cheatsheets/networking/http/headers|HTTP Headers — Quickref]]

## 🔎 How to Use
!!! tip
    Use the search bar for method names, error snippets, or concepts (e.g., `@Transactional`, `N+1`, `Content-Type`).
```

### 5.2 `docs/cheatsheets/index.md` (section landing)

```markdown
---
title: Cheatsheets
summary: Quick references for languages, frameworks, databases, tools, and commands.
---

# ⚡ Cheatsheets

Fast lookup. Minimal theory. Maximum clarity.

## 🧱 Categories
### Languages
- Java: [[cheatsheets/languages/java/core/streams|Streams]] · [[cheatsheets/languages/java/frameworks/spring/annotations|Spring Annotations]]
- Python: [[cheatsheets/languages/python/basics|Basics]]

### Databases
- SQL: [[cheatsheets/databases/sql/basics|Basics]] · [[cheatsheets/databases/sql/joins|Joins]]
- MySQL: [[cheatsheets/databases/mysql/setup/java|Setup (Java)]] · [[cheatsheets/databases/mysql/setup/python|Setup (Python)]] · [[cheatsheets/databases/mysql/queries|Queries]]
- ORM: [[cheatsheets/databases/orm/jpa-java-annotations|JPA Annotations]] · [[cheatsheets/databases/orm/sqlalchemy-python-cheats|SQLAlchemy]]

### Networking
- [[cheatsheets/networking/http/basics|HTTP Basics]] · [[cheatsheets/networking/http/headers|Headers]]

### Tools
- [[cheatsheets/tools/git|Git]] · [[cheatsheets/tools/docker|Docker]] · [[cheatsheets/tools/obsidian-mkdocs-setup|Obsidian + MkDocs Setup]]
```

### 5.3 `docs/concepts/index.md` (section landing)

```markdown
---
title: Concepts
summary: Deep dives into how systems work — theory, trade-offs, and reasoning.
---

# 🧠 Concepts

Where understanding replaces memorization.

## 🧩 Topics
### Backend
- [[concepts/backend/http|HTTP]] · [[concepts/backend/rest-api|REST API]] · [[concepts/backend/caching|Caching]]

### Databases
- [[concepts/databases/normalization|Normalization]] · [[concepts/databases/indexing|Indexing]] · [[concepts/databases/transactions-acid|Transactions (ACID)]]
- ORM: [[concepts/databases/orm/orm-concepts|ORM Concepts]] · [[concepts/databases/orm/jpa-vs-sqlalchemy|JPA vs SQLAlchemy]]

### Frameworks
- [[concepts/frameworks/spring-core|Spring Core]] · [[concepts/frameworks/hibernate|Hibernate]]

### Design
- [[concepts/design/dependency-injection|Dependency Injection]] · [[concepts/design/microservices|Microservices]]
```

**Do you need to define folder structure in `index.md`?**
No. `index.md` is *content*, not configuration. It’s a curated landing page for the folder. The **site structure** is driven by the filesystem and (optionally) the `nav:` in `mkdocs.yml`.

---

## 6) Front-matter templates (copy/paste into pages)

Cheatsheet:

```markdown
---
title: <Title> — Quickref
tags: [cheatsheet, <topic>, <tech>]
summary: One-line reason this exists (lookup while coding).
---
# <Title> — Quickref

> See concepts: [[concepts/...]]
```

Concept:

```markdown
---
title: <Concept Name>
tags: [concept, <domain>]
summary: What it is, why it matters, trade-offs.
---
# <Concept Name>

> See cheatsheet: [[cheatsheets/...]]
```

---

## 7) Local dev & deploy

```bash
# Local preview (auto-reload)
mkdocs serve
# open http://127.0.0.1:8000/

# Deploy to GitHub Pages
mkdocs gh-deploy
# your site: https://<your-username>.github.io/knowledge-vault/
```

---

## 8) FAQ / Gotchas

* **Where should README.md live?** At repo root. It explains the project + how to run/deploy. The site homepage is `docs/index.md`.
* **Do I need `docs_dir: docs`?** No. That’s the default; keeping it implicit is cleaner.
* **What makes `/cheatsheets/` and `/concepts/` routes work?** The `index.md` files in those folders + `use_directory_urls: true`.
* **What if I move files around?** `[[wikilinks]]` are updated by `mkdocs-obsidian` on build. Rebuild after reorganizing.
* **Should I curate `nav:` now?** Early on, skip it. When structure stabilizes, un-comment `nav:` in `mkdocs.yml` to control order and labels.

---

## 9) Mental model (why this works)

* **Separation of concerns:** `README.md` (GitHub), `docs/index.md` (site), `mkdocs.yml` (wiring).
* **Two modes = less friction:** Cheatsheets for speed, Concepts for depth.
* **Wikilinks = future-proof:** Rearrange folders without rewriting links.
* **Indexes = clean URLs:** Section `index.md` gives `/cheatsheets/` and `/concepts/` real landing pages.

Ship it. Then iterate. The vault grows with you, not against you.
