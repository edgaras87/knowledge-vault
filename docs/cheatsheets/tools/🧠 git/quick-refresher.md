---
title: quick-refresher
tags: [git, version-control, vcs, branching, collaboration, troubleshooting]
summary: Git basics, core concepts, commands, branching strategies, undoing changes, working with remotes, and troubleshooting.
---


# 🧠 Git: From Basics to Advanced Workflow Mastery

Git is a **distributed version control system (DVCS)** — it tracks changes to files over time, lets developers collaborate, and makes every clone a complete backup of the repository.  
It’s the tool that powers GitHub, GitLab, and most of modern software development.

---

## ⚙️ 1. What Git Actually Does

Git records **snapshots** of your project (commits), not just diffs.  
Each commit stores the *entire state* of your code at that moment, compressed efficiently.

You can move through history (like checkpoints in a game), branch off to experiment, and merge or rebase to integrate changes.

```bash
# See history
git log --oneline --graph --decorate
````

👉 In short: Git is your project’s **time machine + collaboration layer**.

---

## 🧱 2. Core Concepts

| Concept               | Description                                                                      |
| --------------------- | -------------------------------------------------------------------------------- |
| **Repository (repo)** | A Git-managed folder containing code and version history (`.git` folder inside). |
| **Commit**            | A snapshot of your files at a moment in time.                                    |
| **Branch**            | A movable pointer to a commit — allows parallel development.                     |
| **HEAD**              | Your current working branch (where you are in history).                          |
| **Staging area**      | Where you prepare changes before committing.                                     |
| **Remote**            | A copy of your repo hosted elsewhere (e.g., GitHub).                             |
| **Merge**             | Combines histories from different branches.                                      |
| **Rebase**            | Moves commits onto a new base, creating a linear history.                        |

---

## ⚒️ 3. The Git Workflow (mental model)

```
Working Directory → Staging Area → Local Repository → Remote Repository
```

1. Edit files → `git add`
2. Save snapshot → `git commit`
3. Share/pull changes → `git push` / `git pull`

---

## 🔧 4. Quick Command Reference

### Initialize or Clone

```bash
git init                           # start a new repo
git clone https://github.com/user/repo.git
```

### Inspect

```bash
git status                         # show changed files
git diff                           # show unstaged changes
git log --oneline --graph --decorate
```

### Stage and Commit

```bash
git add file.txt                   # stage file
git add .                          # stage all changes
git commit -m "Add new feature"    # create commit
```

### Branch and Merge

```bash
git branch                         # list branches
git switch -c feature/login        # create + switch
git merge feature/login            # merge into current branch
git branch -d feature/login        # delete local branch
```

### Sync with Remote

```bash
git remote -v                      # show remotes
git fetch                          # get new data, no merge
git pull                           # fetch + merge current branch
git push origin main               # upload local commits
```

---

## 🌿 5. Branching and Collaboration Patterns

**Main workflow:**

```
main
 ├─ feature/login
 ├─ feature/dashboard
 └─ fix/typo
```

**Branch naming convention:**
`feature/…`, `fix/…`, `refactor/…`, `release/…`

**Recommended pattern:**

* `main` — stable release branch
* `develop` — active integration branch
* `feature/*` — short-lived branches
* `hotfix/*` — emergency fixes
* `release/*` — prep for deployment

---

## ⚡ 6. Undoing and Time Travel

```bash
git restore file.txt               # discard unstaged changes
git restore --staged file.txt      # unstage
git checkout <commit> -- file.txt  # restore from old commit
git reset --soft HEAD~1            # undo last commit, keep changes staged
git reset --hard HEAD~1            # undo commit + changes
git revert <commit>                # create new commit that undoes another
```

👉 `reset` rewrites history; `revert` adds a new commit that undoes previous work safely.

---

## 🧩 7. Working with Remotes (GitHub, etc.)

```bash
git remote add origin https://github.com/user/repo.git
git push -u origin main
```

Next time, just:

```bash
git push
git pull
```

**Clone and work:**

```bash
git clone <url>
git switch -c feature/branch
```

**Sync fork or upstream:**

```bash
git remote add upstream https://github.com/source/repo.git
git fetch upstream
git merge upstream/main
```

---

## 🧰 8. Real-World Example (Feature Workflow)

```bash
# Start new work
git switch -c feature/login

# Edit code
git add .
git commit -m "Implement login feature"

# Update main and rebase to stay up-to-date
git fetch origin
git rebase origin/main

# Push branch for review
git push -u origin feature/login

# After merge, clean up
git switch main
git pull
git branch -d feature/login
```

---

## 🧼 9. Maintenance and Cleanup

```bash
git branch -vv                    # see tracking info
git fetch -p                      # prune deleted remote branches
git gc                            # clean up unnecessary files and optimize repo
```

---

## 🧠 10. Common Troubleshooting

| Problem                           | Fix                                               |
| --------------------------------- | ------------------------------------------------- |
| Accidentally committed wrong file | `git reset HEAD~1` then re-commit                 |
| Merge conflict                    | Edit conflicted file → `git add .` → `git commit` |
| Detached HEAD                     | `git switch main` to reattach                     |
| “non-fast-forward” error on push  | Pull first (`git pull --rebase`) then push again  |
| Wrong commit message              | `git commit --amend -m "New message"`             |

---

## 🧩 11. Advanced Tools (for later)

* **Rebase vs Merge:** Rebase keeps history linear, merge preserves branching.
* **Cherry-pick:** apply specific commits between branches.

  ```bash
  git cherry-pick <commit>
  ```
* **Stash:** temporarily save changes.

  ```bash
  git stash
  git stash pop
  ```
* **Hooks:** automate pre-commit checks and CI workflows.
* **Submodules:** include external repos inside your repo.

---

## ✅ Summary

* Git snapshots your code and lets you time-travel safely.
* Commits build your local history, remotes sync it across the team.
* Branches isolate work; merges and rebases integrate it.
* `git status` and `git log` are your best debugging friends.
* Never panic — you can almost always recover history in Git.

---

## 🧭 Next Step Ideas

When you outgrow this refresher:

* Split into `git-basics.md`, `git-advanced.md`, `git-troubleshooting.md`
* Add practical guides for branching strategies or GitHub workflows.

---

📄 **File path suggestion:**

```
docs/
└─ cheatsheets/
   └─ tools/
      └─ git/
         └─ quick-refresher.md
```






















---

## 💻 12. Git in Your IDE (JetBrains & VS Code)

Git isn’t just a command-line tool — every modern IDE wraps it into a visual workflow.  
But it helps to understand what each action *actually does* under the hood.

---

### 🧩 JetBrains IDEs (IntelliJ IDEA, PyCharm, etc.)

| Action | What It Actually Does |
|--------|------------------------|
| **Commit (Ctrl+K)** | Stages & commits changes locally. Equivalent to `git add` + `git commit`. |
| **Push (Ctrl+Shift+K)** | Uploads commits to the remote (`git push`). |
| **Update Project (Ctrl+T)** | Fetch + merge (or rebase, if configured). |
| **Rebase Onto Upstream** | Equivalent to `git rebase origin/main`. Keeps history clean. |
| **Show History (Alt+9 → Log tab)** | Visual `git log` + branch graph. |
| **Cherry-pick Commit** | Applies selected commit(s) onto your branch (`git cherry-pick`). |
| **Shelve Changes** | JetBrains' own version of `git stash` — great for quick context switches. |

👉 Pro tip: JetBrains auto-detects Git roots. If a folder contains `.git`, it treats it as a repo automatically.

**Good workflow habit inside JetBrains:**
1. Regularly commit small, meaningful chunks.
2. Rebase before pushing to keep branches fast-forwardable.
3. Always inspect your commit diff before confirming.

---

### 🧠 VS Code Integration

VS Code’s Source Control panel is a friendly wrapper around core Git commands.

| Icon | Action | Git Equivalent |
|------|---------|----------------|
| ✓ | Commit | `git commit` |
| ⬆️ | Push | `git push` |
| ⬇️ | Pull | `git pull` |
| 🔁 | Sync Changes | Pull + Push |
| ⊕ | Stage Change | `git add` |
| 🗑️ | Discard Change | `git restore` |

**Extensions worth adding:**
- **GitLens** → deep history, blame, branch insights.
- **Git Graph** → visual branching and merges.
- **GitHub Pull Requests** → manage PRs directly in VS Code.

---

## ✍️ 13. Writing Great Commit Messages

Clear commit messages are part of professional hygiene.  
A Git log should read like a timeline of meaningful decisions, not noise.

**Format (conventional style):**
```

<type>(scope): short summary

[optional body]
[optional footer]

```

**Examples:**
```

feat(auth): add JWT token validation
fix(api): correct null pointer on user fetch
docs(readme): add setup instructions
refactor(core): simplify cache layer

```

**Common types:**

| Type | Meaning |
|------|----------|
| `feat` | new feature |
| `fix` | bug fix |
| `docs` | documentation change |
| `refactor` | code restructure without behavior change |
| `test` | adding or fixing tests |
| `chore` | maintenance or tooling changes |

👉 A good commit message should explain **why** a change exists, not just **what** changed.

---

## 🧩 14. Practical Git + IDE Workflow Example

Here’s how a typical developer loop looks:

1. **Start new work**
```

git switch -c feature/login

```
*(Or use JetBrains “New Branch” button.)*

2. **Code → test → commit**
- Stage meaningful changes
- Commit with a clear message:  
  `feat(login): add session management`

3. **Stay up to date**

```
git fetch origin
git rebase origin/main
```

*(Or “Rebase onto Main” in IDE.)*

4. **Push for review**


git push -u origin feature/login


*(Or “Push” in IDE.)*

5. **After PR merge**


git switch main
git pull
git branch -d feature/login


*(Or “Delete Branch” safely in IDE.)*

---

## 🧼 15. Tips for Clean Repositories

- Keep `.gitignore` tight — no IDE caches or `.venv/` folders.
- Avoid committing binaries, logs, or secrets.
- Use `.gitattributes` to manage line endings across systems.
- Squash small commits before merging to keep history readable:

```bash
git rebase -i HEAD~5
````

* Use `git tag` for versioning:

  ```bash
  git tag -a v1.0.0 -m "Initial release"
  git push origin v1.0.0
  ```

---

## ✅ Final Notes

* The IDE is just a lens — Git remains the same underneath.
* Always review the diff before committing.
* Favor **small, focused commits** over massive all-in-one pushes.
* Treat your Git history as a story — future you will thank you.

