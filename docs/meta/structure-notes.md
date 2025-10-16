# 🧩 Structure Notes — Cheatsheets Organization

**Topic:** How to handle the growth of the `tools/` section in cheatsheets.

---

### Current Approach

The **`tools/`** folder starts **flat** — everything goes here:

* `git.md`, `docker.md`, `nginx.md`, `obsidian-mkdocs-setup.md`, etc.

At this stage, the goal is **speed and clarity**, not perfect hierarchy.
Don’t overthink categories too early.

```
cheatsheets/
└─ tools/
   ├─ git.md
   ├─ docker.md
   ├─ nginx.md
   ├─ obsidian-mkdocs-setup.md
```

---

### Planned Evolution

As the collection grows and themes start to emerge (e.g., multiple networking or DevOps tools),
split `tools/` into **domain-based subfolders**:

```
cheatsheets/
└─ tools/
   ├─ devops/
   │  ├─ docker.md
   │  ├─ ansible.md
   │  └─ kubernetes.md
   ├─ system/
   │  ├─ systemd.md
   │  ├─ cron.md
   │  └─ journalctl.md
   ├─ networking/
   │  ├─ nginx.md
   │  └─ certbot.md
   └─ productivity/
      ├─ git.md
      ├─ mkdocs.md
      └─ obsidian.md
```

---

### Guiding Principles

* **Start flat, grow organically.** Don’t create folders for one file.
* **When 3+ tools cluster by domain**, introduce a subfolder.
* **Cheatsheets** are *how to use a thing.*
  **Concepts** are *why the thing works that way.*
* The goal isn’t a perfect taxonomy — it’s a **navigable map of thought** that evolves with experience.

---

### Notes

This convention prevents early complexity while ensuring long-term scalability.
It reflects the “knowledge-vault” philosophy: *build what you need today, structure it when it starts to breathe.*

