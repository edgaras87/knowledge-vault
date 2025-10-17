---
title: Remote ↔ Local
date: 2025-10-17
tags: 
    - git
    - cheatsheet
    - version-control
    - devops
    - workflow
    - tools
summary: Git commands cheat sheet for syncing local repositories with remote ones.
aliases:
  - Remote ↔ Local Cheat Sheet
---

# 📝 Git Local ↔ Remote Cheat Sheet

---

## 🔑 Core ideas

* **clone** → make a new local folder from remote  
* **init** → make current folder a repo  
* **remote** = nickname + URL (`origin`)  
* **upstream** = default remote branch your local branch tracks  

---

## 🚀 Common Scenarios

### 1. Start from Remote → Local

```bash
git clone <REMOTE-URL> [folder]
cd [folder]
```

✅ Now just `git pull` and `git push`

---

### 2. Start from Local → Remote

```bash
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin <REMOTE-URL>
git push -u origin main
```

> The `-u` flag sets the **upstream** for your branch.
> It tells Git: “`main` should track `origin/main` from now on.”

---

### 3. Already Have Both (Sync Them)

```bash
git remote -v
git fetch origin
git switch main
git pull --rebase
git push
```

If unrelated histories:

```bash
git pull --rebase --allow-unrelated-histories
```

---

## 🌍 Understanding “Upstream”

The **upstream branch** is what your local branch “tracks.”
When you set it, `git pull` and `git push` automatically know which remote branch to sync with — no need to retype `origin main` every time.

```bash
git branch -vv        # see tracking info
git push -u origin main   # set upstream during first push
git branch --set-upstream-to=origin/main main  # set manually
```

After setting upstream:

```bash
git pull     # pulls from origin/main automatically
git push     # pushes to origin/main automatically
```

If you ever clone a repo, upstreams are usually set for you.
You’ll only need to set them manually when creating new branches or pushing from a freshly inited local repo.

> Think of it this way:
> `origin` is the *remote name* (like a server nickname).
> `origin/main` is the *remote branch*.
> The *upstream* is your local branch’s connection to it.

---

## 📂 Folder Rules

* `git clone <url>` → makes a new folder
* `git clone <url> myfolder` → makes *myfolder*
* `git clone <url> .` → current empty folder
* **Always run Git inside the repo folder**

---

## 🤔 Which First?

* **Solo / quick test:** local → push
* **Team / templates / CI:** remote → clone

---

## ⚡ Handy Commands

```bash
git remote -v                    # list remotes
git push -u origin main          # first push, set upstream
git pull --rebase                # cleaner pulls
git remote set-url origin <url>  # change remote
git branch -vv                   # show tracking branches
```

---

## ⚠️ Pitfalls

* Remote has README → `git pull --rebase --allow-unrelated-histories`
* Wrong branch name → `git branch -M main`
* Tracked junk files → fix `.gitignore`, then:

  ```bash
  git rm -r --cached .
  git add .
  git commit -m "clean tracked files"
  ```

---

## 🗂️ Decision Tree

* **Have remote?** → `git clone <url>`
* **Have local only?** → `git init` → `git push -u origin main`
* **Both exist?** → `git fetch` → `git pull --rebase` → `git push`

---

👉 This is the minimal flow you’ll use 99% of the time.

```

---

### Why this matters

Without an upstream, every `git pull` or `git push` must specify remote + branch manually (`git push origin main`). Once set, you can just type `git push`. That’s why `-u` is quietly one of the most powerful flags in Git — it wires your local branch into the network.
