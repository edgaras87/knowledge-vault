---
title: .venv
tags:
    - python
    - environments
    - venv
    - virtualenv

summary: A quick reference on Python virtual environments using `venv`.
aliases:
  - venv Cheatsheet
---




# 🧠 Python Virtual Environments (`venv`) — Cheatsheet

### 🧩 What It Is

`venv` means *virtual environment*.
It’s a **self-contained Python environment** that lives inside your project folder.
Think of it as a *sandbox* where you can install Python packages without messing up your system or other projects.

Each `venv`:

* Has its own **Python interpreter** (copied from your system’s).
* Has its own **`site-packages`** folder (where libraries get installed).
* Ignores all global packages unless told otherwise.

---

### 🎯 Why Use It

Without `venv`, if you install libraries like this:

```bash
pip install mkdocs
```

they go into your *global Python* — polluting everything.
Soon you’ll have version conflicts and dependency chaos between projects.

With `venv`:

* Every project has **isolated dependencies**.
* You can safely use different library versions per project.
* You can delete or recreate it anytime — your system stays clean.

---

### ⚙️ How To Create and Use

**1. Create a venv**

```bash
python3 -m venv .venv
```

This makes a folder `.venv` with a fresh Python setup inside.

**2. Activate it**

| System               | Command                      |
| -------------------- | ---------------------------- |
| Linux / macOS        | `source .venv/bin/activate`  |
| Windows (cmd)        | `.venv\Scripts\activate`     |
| Windows (PowerShell) | `.venv\Scripts\Activate.ps1` |

When active, your shell prompt changes — usually you’ll see `(.venv)` in front.

**3. Install libraries**

```bash
pip install mkdocs mkdocs-material mkdocs-obsidian
```

They’ll install **inside `.venv` only**, not globally.

**4. Freeze (optional)**
If you want to share exact versions with others:

```bash
pip freeze > requirements.txt
```

Then anyone can recreate your setup with:

```bash
pip install -r requirements.txt
```

**5. Deactivate**

```bash
deactivate
```

---

### 🧹 In `.gitignore`

You always exclude `.venv/`:

```
.venv/
```

Because it’s bulky, system-specific, and can be recreated from `requirements.txt`.

---

### 🧠 Quick Summary Table

| Task                | Command                           | Notes                         |
| ------------------- | --------------------------------- | ----------------------------- |
| Create              | `python3 -m venv .venv`           | Makes environment in `.venv/` |
| Activate            | `source .venv/bin/activate`       | Start using it                |
| Deactivate          | `deactivate`                      | Return to system Python       |
| Install packages    | `pip install <package>`           | Only affects this project     |
| List packages       | `pip list`                        | Shows installed inside venv   |
| Export dependencies | `pip freeze > requirements.txt`   | For version tracking          |
| Reinstall from file | `pip install -r requirements.txt` | Recreates setup               |

---

### 🧩 Mental Model

Imagine your computer is a city.
Each Python project is its own *apartment*.
`venv` gives each apartment:

* Its own kitchen (libraries)
* Its own utilities (interpreter)
* But shares the city’s infrastructure (OS-level Python)

So if one apartment burns its kitchen down (dependency conflict), the others stay safe.

---

# ⚠️ `venv` Troubleshooting & Common Pitfalls

### 🧩 1. “I installed a package, but Python says ‘ModuleNotFoundError’”

**Cause:** You forgot to activate your virtual environment.
**Fix:**
Activate it *before* installing or running:

```bash
source .venv/bin/activate
```

You’ll know it’s active when you see `(.venv)` before your prompt.

---

### 🐍 2. “I have multiple Python versions — which one is used?”

Each venv is tied to the Python version that created it.
Example:

```bash
python3.11 -m venv .venv
```

means your venv uses **Python 3.11**, even if your system later upgrades to 3.12.
If you remove or change your system Python, that venv might break.

**Tip:** Always note your Python version in your README.

---

### 💀 3. “The venv folder got corrupted or too big”

Don’t panic — just **delete `.venv/`** and recreate it:

```bash
rm -rf .venv
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

That’s it. Your clean sandbox returns, good as new.

---

### 🧠 4. “MkDocs (or some tool) not found even though I installed it”

That means the tool was installed inside the venv, but your shell isn’t using that path.
**Fix:** activate the venv before running:

```bash
source .venv/bin/activate
mkdocs serve
```

or run it explicitly from inside:

```bash
.venv/bin/mkdocs serve
```

---

### 💼 5. “Do I need to commit `.venv` to Git?”

**Never.**
Just commit your `requirements.txt`.
`.venv` is like your temporary workbench — everyone can rebuild it from the list of dependencies.

---

### ⚙️ 6. “Can I have more than one venv?”

Absolutely.
You can have one per project, or even multiple inside one project for different experiments:

```
.venv/
.venv-py311/
.venv-py312/
```

Just activate whichever one you need.

---

### 🔄 7. “I closed the terminal — do I have to activate again?”

Yes.
The activation is per shell session.
Once you close it, your environment resets, and you’ll need to:

```bash
source .venv/bin/activate
```

again when reopening your project.

---

### 🧠 Quick Diagnosis Checklist

If something isn’t working:

1. Is your prompt showing `(.venv)`?
   → If not, activate it.
2. Does `which python` or `where python` show `.venv/bin/python`?
   → If not, wrong environment.
3. Did you reinstall dependencies after deleting `.venv`?
   → Run `pip install -r requirements.txt`.

---

### 💡 Golden Rule

When in doubt:

> Delete `.venv`, recreate it, reinstall.
> It’s faster than debugging a broken environment.


---

# ⚙️ Advanced `venv` Tricks & Integrations

### 🧠 1. Auto-activate in VS Code

VS Code has native support for virtual environments.

1. Open your project folder in VS Code (the one with `.venv/`).
2. Open Command Palette → `Python: Select Interpreter`.
3. Choose the one that points to `.venv/bin/python` (or `.venv\Scripts\python.exe` on Windows).

Now every terminal inside VS Code **auto-activates the venv**, and “Run” or “Debug” commands use it automatically.

**Tip:** Add this to your project’s `.vscode/settings.json` (optional, but nice):

```json
{
  "python.defaultInterpreterPath": ".venv/bin/python"
}
```

---

### 🐚 2. Auto-activate when entering folder (shell-level trick)

If you live in your terminal, you can make the venv activate automatically whenever you `cd` into the project.

#### For **bash** or **zsh**:

Append to your `~/.bashrc` or `~/.zshrc`:

```bash
# Auto-activate venv when entering project folder
cd() {
  builtin cd "$@" || return
  if [ -f ".venv/bin/activate" ]; then
    source .venv/bin/activate
  fi
}
```

That makes the environment self-starting — no more “forgot to activate” moments.

---

### 🐳 3. Use venv in Docker (for consistent builds)

You usually don’t need `venv` *inside* Docker because Docker itself isolates environments —
but it’s sometimes useful when developing locally *and* deploying via container.

A typical lightweight pattern:

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY . .
RUN python3 -m venv /venv
ENV PATH="/venv/bin:$PATH"

RUN pip install --upgrade pip && pip install -r requirements.txt
CMD ["mkdocs", "serve", "-a", "0.0.0.0:8000"]
```

This way, the container runs everything inside `/venv`, giving full reproducibility between your laptop and the container.

---

### 🔄 4. Use venv in CI/CD (GitHub Actions, GitLab, etc.)

Virtual environments integrate beautifully in automation.
Example GitHub Actions job for MkDocs:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: python -m venv .venv
      - run: source .venv/bin/activate && pip install -r requirements.txt
      - run: source .venv/bin/activate && mkdocs build
```

That ensures each run is clean, reproducible, and isolated — no dependency drift over time.

---

### 🧩 5. Use venv executables directly (no activation needed)

Sometimes automation scripts shouldn’t depend on interactive activation.

You can run any installed tool directly from inside `.venv/bin`:

```bash
.venv/bin/mkdocs serve
```

That works even if the venv isn’t “activated.”
It’s cleaner in scripts, cron jobs, or Dockerfiles.

---

### 🧹 6. Clean up and reset your environment fast

When you want a fresh start:

```bash
rm -rf .venv site/ __pycache__/
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

That gives you a sterile lab again, like hitting “reset” on a scientific experiment.

---

### 🧠 Rule of Thumb for Power Users

* **Local dev** → always use `.venv`
* **Dockerized app** → system Python inside container is fine, or one global `/venv`
* **CI/CD** → recreate venv per run for reliability

---

Next level from here would be **environment management tools** like:

* `pip-tools` → for precise dependency pinning
* `poetry` → combines venv + dependency management + packaging
* `pyenv` → handles multiple Python versions on one system

