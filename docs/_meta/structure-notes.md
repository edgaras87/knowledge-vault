---
title: Structure Notes 
date: 2025-10-21 
summary: Guidelines for organizing cheatsheets and concepts sections in the knowledge vault.
---


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


Here’s a matching note for your **Concepts** section, written in the same tone and structure as your *Structure Notes — Cheatsheets Organization* one:

---


```aiignore





```


---

# 🧠 Structure Notes — Concepts Organization

**Topic:** How to handle the growth and structure of the `concepts/` section.

---

### Current Approach

The **`concepts/`** folder starts **flat** — every new idea or explanation goes here directly:

* `http.md`, `tcp-ip.md`, `oop.md`, `rest-vs-graphql.md`, etc.

At the beginning, the goal is **clarity and speed** of capture.
Don’t pre-plan categories before ideas have weight.

```
concepts/
├─ http.md
├─ tcp-ip.md
├─ oop.md
└─ rest-vs-graphql.md
```

---

### Planned Evolution

As understanding deepens and related topics multiply, group them into **domain-based subfolders**.

```
concepts/
├─ networking/
│  ├─ http/
│  │  ├─ http-basics.md
│  │  ├─ headers.md
│  │  ├─ methods.md
│  │  ├─ status-codes.md
│  │  └─ caching.md
│  ├─ tcp-ip.md
│  ├─ dns.md
│  └─ ssl-tls.md
├─ web/
│  ├─ rest-vs-graphql.md
│  ├─ api-versioning.md
│  ├─ cookies-vs-tokens.md
│  └─ cors.md
├─ programming/
│  ├─ oop.md
│  ├─ functional-programming.md
│  ├─ async-vs-threading.md
│  └─ design-patterns.md
└─ os/
   ├─ processes-vs-threads.md
   ├─ file-descriptors.md
   └─ memory-management.md
```

---

### Guiding Principles

* **Start flat, grow when patterns emerge.**
  A single note doesn’t deserve a folder — but a cluster does.
* **Concepts explain systems, not tools.**
  They answer *why and how something works* rather than *how to use it*.
* **Mirror real-world domains.**
  Networking, Web, Programming, and OS — a conceptual map of how the stack fits together.
* **Keep modular granularity.**
  Break large concepts (like HTTP) into smaller, linkable notes once needed.

---

### Notes

This pattern scales from beginner notes to deep theory without collapse into chaos.
Where *cheatsheets* are practical quick wins, *concepts* are the slow architecture of understanding — ideas that deserve to interlink and mature.

It keeps your vault growing like a **mind with memory**, not a folder full of files.

