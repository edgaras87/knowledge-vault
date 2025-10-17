---
title: Containers — Future Blueprint
date: 2025-10-17
summary: A scalable folder structure for documenting Docker, Podman, and container ecosystem notes — balancing conceptual overviews, commands, templates, and orchestration.
---

# Containers/ Blueprint

---

## Purpose

The `containers/` domain organizes everything related to container engines, images, runtime behavior, orchestration, and ecosystem tooling.

It separates **engine-specific docs (Docker, Podman)** from **cross-engine concepts** like images, networks, volumes, and orchestration (Compose/Kubernetes).

This prevents duplication and keeps the collection evolving smoothly as new tools (Buildah, Skopeo, Finch, Rancher, etc.) join the ecosystem.

---

## Folder layout (future-proof)

```

cheatsheets/
└─ containers/
├─ _index.md                       # High-level overview (OCI standard, purpose, tools)
├─ engines/
│  ├─ docker.md                    # Install, daemon, BuildKit, Compose v2
│  ├─ podman.md                    # Install, rootless, pods, Quadlet, docker.sock compat
│  └─ notes.md                     # Common engine quirks or comparisons
├─ images/
│  ├─ building.md                  # Dockerfile / Containerfile, multi-arch, BuildKit vs Buildah
│  ├─ scanning.md                  # Image scanning (trivy, grype)
│  └─ caching.md                   # Layers, pruning, build cache
├─ registries/
│  ├─ auth.md                      # Login, creds store, auth.json, helper tools
│  └─ push-pull.md                 # Tagging, pushing, pulling, skopeo copy
├─ runtime/
│  ├─ run-basics.md                # run/start/stop/logs/exec equivalents
│  ├─ pods-vs-compose.md           # When to use Pods (Podman) vs Compose (Docker)
│  └─ systemd-quadlet.md           # systemd integration for containers
├─ networking/
│  ├─ basics.md                    # bridge/host/macvlan modes, ports
│  └─ dns.md                       # Container DNS, resolv.conf, hosts mapping
├─ storage/
│  ├─ volumes.md                   # Named/bind mounts, persistence, SELinux :Z flags
│  └─ layers.md                    # OverlayFS, pruning, cleanup strategies
├─ orchestration/
│  ├─ compose.md                   # Compose schema, common stack patterns
│  ├─ kubernetes.md                # Core K8s concepts, manifests, migration from Compose
│  └─ quadlet.md                   # Podman Quadlet + systemd unit patterns
├─ templates/
│  ├─ compose/                     # Prebuilt service templates (MySQL, Redis, Nginx, etc.)
│  ├─ quadlet/                     # Systemd unit templates for rootless containers
│  └─ k8s/                         # Optional: minimal Deployment/Service manifests
├─ security/
│  ├─ rootless.md                  # User namespaces; Podman’s approach
│  ├─ capabilities.md              # cap drop/add, seccomp, SELinux/AppArmor
│  └─ supply-chain.md              # SBOMs, signatures, provenance
├─ troubleshooting/
│  ├─ inspect-and-logs.md          # inspect, logs, events, journald
│  └─ cleanup.md                   # prune, orphan cleanup, storage reset
└─ _meta/
└─ blueprint.md                 # (this file)

````

---

## File naming & style rules

- **One concept per file.**
  Keep each note focused — e.g., `volumes.md`, `building.md`, `run-basics.md`.
- **Use kebab-case** filenames (consistent with URLs).
- **Front matter metadata** aligns with the rest of your cheatsheets:

```md
---
title: Podman — Rootless Mode
tags: [containers, podman, rootless]
summary: Understanding rootless containers and user namespaces in Podman.
aliases:
---
```

* **Titles**: “Engine — Concept” or “Concept — Focus”

  * Examples: `Docker — Daemon & Compose`, `Images — Building`, `Networking — Basics`.
* **Cross-link** pages frequently (e.g., link from `images/building.md` to `registries/push-pull.md`).

---

## Optional: `README.md` Index per subfolder

Example: `containers/runtime/README.md`

```md
---
title: Container Runtime — Index
tags: [containers, runtime]
summary: Container lifecycle, logs, and system integration.
---

- [Run Basics](./run-basics.md)
- [Pods vs Compose](./pods-vs-compose.md)
- [Systemd & Quadlet](./systemd-quadlet.md)
```

---

## Quick rename / migration guidance

* Existing single-page Docker notes → move to `engines/docker.md`
* Short templates (e.g., MySQL, Redis, Nginx Compose files) → `templates/compose/`
* Rootless systemd or Quadlet examples → `templates/quadlet/`
* Conceptual guides (volumes, networking, etc.) → respective subfolders

---

## Future directions (for when you scale)

* Add `containers/tools/` for **Buildah**, **Skopeo**, **Trivy**, **Grype**.
* Add `containers/integration/` for **CI/CD**, **GitHub Actions**, **image signing**.
* Add `containers/_meta/summary.md` — yearly reflection on what grew, what merged, what’s deprecated.

---

## Core principle

> Organize by **concept**, not by engine.
> Engines change; concepts survive.

Every folder should answer one enduring question:

* **Images/** — How do we build and move container blueprints?
* **Runtime/** — How do we run and manage containers locally?
* **Orchestration/** — How do we run multiple containers together?
* **Security/** — How do we trust, sign, and isolate them?
* **Troubleshooting/** — How do we inspect and fix them?

That’s how you future-proof `containers/` without overcomplicating it today.

