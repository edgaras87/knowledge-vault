# 🧠 Knowledge Vault

👉 Live docs: https://edgaras87.github.io/knowledge-vault/

---

This vault is divided into **two major sections** — `cheatsheets/` and `concepts/`.

Each serves a different mindset:

- **Cheatsheets** → Quick practical reference. Short, actionable, syntax or API focused.  
  (Example: `Stream.map()` usage, `JPA annotations`, `MySQL setup for Java`)
- **Concepts** → Explanatory and theoretical. How and *why* something works.  
  (Example: `ORM concepts`, `HTTP lifecycle`, `Dependency Injection`)

Both are written in Markdown, interlinked with `[[WikiLinks]]`, and rendered via **MkDocs + Material**.

---

## 🚀 Preview Locally

You’ll need **Python 3.8+** and `pip`.


#### 1. Clone the repo

```bash
git clone https://github.com/edgaras87/knowledge-vault.git
cd knowledge-vault
```

#### 2. (Optional) Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
```

#### 3. Install dependencies

```bash
pip install mkdocs mkdocs-material mkdocs-obsidian mkdocs-git-revision-date-localized-plugin
```

#### 4. Run the local dev server

```bash
mkdocs serve
```

Then open your browser at **[http://127.0.0.1:8000/](http://127.0.0.1:8000/)**.

> 💡 Tip: Add .venv/ and site/ to your .gitignore to keep builds and virtual environments out of Git.

---

## 🧩 Project Structure

```
knowledge-vault/
├─ mkdocs.yml          # MkDocs configuration
├─ README.md           # You’re reading this — repo overview (for GitHub)
└─ docs/               # Actual documentation content
   ├─ index.md         # Site homepage
   ├─ cheatsheets/     # Quick references (language APIs, tools, patterns)
   │   ├─ languages/
   │   │   ├─ java/
   │   │   └─ python/
   │   ├─ databases/
   │   └─ tools/
   └─ concepts/        # In-depth explanations and theory
       ├─ backend/
       ├─ databases/
       └─ design/
```

---

## 🧠 How It Works

* `mkdocs.yml` — the **blueprint**: defines site title, theme, plugins, and navigation.
* `docs/index.md` — the **homepage** for the generated site.
* `README.md` — the **GitHub landing page**, separate from the live docs.

MkDocs reads from `/docs` → builds a static website into `/site`.

---

## 🌐 Publish Online

Once you’re happy with your content, deploy with one command:


```bash
mkdocs gh-deploy
```

This builds the site and publishes it automatically to
**https://<your-username>.github.io/knowledge-vault/**

> (The gh-deploy command automatically creates and updates the gh-pages branch used by GitHub Pages.)

---

## 🧭 Editing Flow

1. Add or edit `.md` files inside `docs/`.
2. Run `mkdocs serve` to preview changes live.
3. Commit and push to GitHub.
4. Deploy with `mkdocs gh-deploy` (optional).

---

## 🧩 Notes

* MkDocs automatically rebuilds whenever you save a file.
* Obsidian-style links like `[[concepts/backend/http]]` work thanks to `mkdocs-obsidian`.
* Section `index.md` files (like `docs/concepts/index.md`) act as clean landing pages.

## 📄 Docs & Config

**Quick mental model**
- `README.md` → for GitHub (repo intro, run/deploy)
- `docs/index.md` → site homepage (reader-facing)
- `mkdocs.yml` → wiring (theme, plugins, nav)

**Jump to files**
- Site homepage: [`docs/index.md`](docs/index.md)
- MkDocs config (commented): [`mkdocs.yml`](mkdocs.yml)
- Full setup & reasoning: [`docs/cheatsheets/tools/obsidian-mkdocs-setup.md`](docs/cheatsheets/tools/obsidian-mkdocs-setup.md)


---

## 📘 Conventions

**Naming**
- Use lowercase-with-hyphens for filenames (`rest-controller.md`, not `RestController.md`).
- Use descriptive names over generic ones (`sqlalchemy-python-cheats.md`, not `orm2.md`).
- Each folder can have its own `index.md` to serve as a landing page.

**Tags**
- Optionally tag pages for cross-searching:
  - `tags: [cheatsheet]`
  - `tags: [concept]`
- Combine tags for filters like `tag:cheatsheet path:mysql`.

**Links**
- Use Obsidian wikilinks: `[[cheatsheets/databases/orm/jpa-java-annotations]]`
- They’ll auto-convert to working links in MkDocs builds.

**Cross-linking**
- Cheatsheets should link “up” to related concepts:  
  `See [[concepts/databases/orm/orm-concepts]]`
- Concepts can link “down” to language implementations:  
  `Example: [[cheatsheets/databases/orm/sqlalchemy-python-cheats]]`

**Structure philosophy**
- Organize by **domain** first, **language** second.
- Avoid duplication — if knowledge spans languages (e.g., MySQL), keep it under one root and make subfolders per language/tool.
- Add new domains naturally (e.g. `frontend/`, `devops/`, `security/`) as your vault grows.

---

## 🚀 Purpose

This vault acts as a **personal developer wiki**:  
- A quick reference when coding (cheatsheets)  
- A deep knowledge archive for review and mastery (concepts)  
- A publishable site via MkDocs Material

Keep it consistent, cross-linked, and evolving.

---

_Made with curiosity and caffeine ☕ by Edgaras._



