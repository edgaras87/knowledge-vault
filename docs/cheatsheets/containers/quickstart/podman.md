---
title: 🚀 Podman Quickstart
date: 2025-10-17
tags: 
    - podman
    - containers
    - devops
    - orchestration
    - rootless
summary: A concise introduction to Podman, its architecture, core concepts, commands, rootless containers, pods, systemd integration, and comparison with Docker.
aliases:
    - Podman Quickstart

---


# 🚀 Simple Guide to `Podman` and Containers

---

## 🔧 What Podman Is

* **Podman** (“Pod Manager”) is a **container engine** — just like Docker — but it runs **without a daemon**.
  Instead of one background service (`dockerd`) managing everything, Podman runs containers as **regular Linux processes** owned by the user who starts them.

* It fully supports the **OCI (Open Container Initiative)** standards, meaning:

  * Docker and Podman use the same image format.
  * You can pull from the same registries (`docker.io`, `quay.io`, `ghcr.io`, etc.).
  * Images built by one work in the other.

* Podman can manage:

  * Single containers
  * Groups of containers (called **Pods**)
  * Containers as **systemd services** via **Quadlet**

---

## 🧱 The Big Architectural Difference

| Concept            | Docker                                 | Podman                                                        |
| ------------------ | -------------------------------------- | ------------------------------------------------------------- |
| Daemon             | Central background process (`dockerd`) | **Daemonless** — every container is a direct child process    |
| Privileges         | Root daemon (even for non-root users)  | **Rootless by default** — containers run under your own UID   |
| Socket             | `/var/run/docker.sock`                 | Optional: `/run/user/$UID/podman/podman.sock`                 |
| Compose            | Built-in `docker compose`              | External tool: `podman-compose`                               |
| Pods               | Docker: only in Kubernetes             | **First-class feature** — group containers like mini-K8s pods |
| System integration | Docker runs as service                 | Podman integrates natively with **systemd** via Quadlet       |

In short: Docker = one big engine. Podman = small, self-contained tools running directly on Linux.

---

## 🧠 Core Podman Concepts (same as Docker)

* **Image**

  * Template for containers, read-only, immutable.
  * Example: `podman pull mysql:8.4`

* **Container**

  * A running instance of an image.
  * Starts quickly, isolated by namespaces and cgroups.
  * Example: `podman run -d --name mysql mysql:8.4`

* **Volume**

  * Persistent data storage — survives container deletion.
  * Example: `podman volume create mysql-data`

* **Ports**

  * Expose container ports to the host.
  * Example: `-p 8080:80` → access on `localhost:8080`.

* **Containerfile**

  * Same as Dockerfile, just name-neutral.
  * Example: `podman build -t myapp .`

* **Compose**

  * Use `podman-compose` (separate package) to run multi-container stacks.

---

## ⚙️ Podman Basics — Command Equivalents

| Task             | Docker                  | Podman                  |
| ---------------- | ----------------------- | ----------------------- |
| Run container    | `docker run -d nginx`   | `podman run -d nginx`   |
| List containers  | `docker ps`             | `podman ps`             |
| Stop container   | `docker stop nginx`     | `podman stop nginx`     |
| Remove container | `docker rm nginx`       | `podman rm nginx`       |
| View logs        | `docker logs nginx`     | `podman logs nginx`     |
| Inspect          | `docker inspect nginx`  | `podman inspect nginx`  |
| View images      | `docker images`         | `podman images`         |
| Build image      | `docker build -t app .` | `podman build -t app .` |

They’re nearly identical — Podman was intentionally built to be CLI-compatible.

---

## 🧍 Rootless Containers (Podman’s Superpower)

By default, Podman runs containers **without root privileges** — each container runs inside a **user namespace**, isolated from the real system.

* You don’t need `sudo`.
* You can run multiple user-isolated container sets on the same machine.
* Security improves because even a breakout from the container stays inside your user sandbox.

Example:

```bash
podman run --rm -it alpine sh
```

→ Runs as your own user, not root.
Inside the container, you *appear* as root — but it’s **mapped** to your normal UID on the host.

---

## 🧩 Pods — Built-in Kubernetes-Style Grouping

A **Pod** is a group of containers that share the same network and storage namespace.

Example:

```bash
podman pod create --name webpod -p 8080:80
podman run -d --pod webpod nginx
podman run -d --pod webpod redis
```

Now both `nginx` and `redis` share the same network — just like a Kubernetes pod.

You can inspect:

```bash
podman pod ps
podman pod inspect webpod
```

Use pods when you want tight coupling between containers (e.g., app + sidecar).

---

## 🧰 Systemd Integration — Quadlet

Podman talks natively with **systemd**, which means your containers can become permanent user or system services.

You don’t write `systemctl start podman` — instead, you drop a `.container` or `.pod` file in your systemd config.

Example: `~/.config/containers/systemd/mysql.container`

```ini
[Container]
Image=mysql:8.4
PublishPort=3306:3306
Env=MYSQL_ROOT_PASSWORD=root
Volume=mysql-data:/var/lib/mysql

[Install]
WantedBy=default.target
```

Enable it:

```bash
systemctl --user daemon-reload
systemctl --user enable --now mysql.service
```

Podman + Quadlet replaces the Docker daemon entirely — systemd manages the container lifecycle.

---

## 🪄 Compose (Multi-Container Dev Environments)

Podman doesn’t bundle Compose by default, but you can install `podman-compose`.

It uses **the same `docker-compose.yml` files**.

Example:

```bash
podman-compose -f docker-compose.yml up -d
podman-compose down
```

That means all your old Compose projects (MySQL + backend + Nginx) run untouched.
Behind the scenes, it creates pods for you automatically.

---

## 🛰️ Networking (Under the Hood)

* Podman uses **CNI plugins** to manage networking.
* Containers on the same pod share the same network namespace.
* For external exposure, use `-p hostPort:containerPort` just like Docker.

Example:

```bash
podman run -d -p 8080:80 nginx
```

To connect multiple containers:

```bash
podman network create appnet
podman run -d --network appnet --name db mysql:8.4
podman run -d --network appnet --name app backend:latest
```

They can talk to each other via `db:3306`, just like in Docker networks.

---

## 📦 Volumes and Storage

Persistent data works the same way as Docker, but stored under your user directory for rootless containers.

```bash
podman volume create mysql-data
podman run -v mysql-data:/var/lib/mysql mysql:8.4
```

List and inspect volumes:

```bash
podman volume ls
podman volume inspect mysql-data
```

---

## 🔐 Security & Isolation

* Rootless mode prevents host compromise.
* Supports **SELinux**, **AppArmor**, and **seccomp** natively.
* You can drop capabilities, restrict syscalls, or limit resources:

  ```bash
  podman run --cap-drop=ALL --read-only nginx
  ```
* Default network isolation is tighter than Docker’s, but can be configured.

---

## 🧹 Maintenance & Cleanup

```bash
podman ps -a
podman images
podman system prune -a
```

Equivalent to Docker — just without daemon interaction.

---

## ⚡ Quick Comparison Summary

| Feature             | Docker                 | Podman                                  |
| ------------------- | ---------------------- | --------------------------------------- |
| Daemon              | Required (`dockerd`)   | None                                    |
| Rootless mode       | Partial                | Default                                 |
| Compose             | Built-in               | Separate tool (`podman-compose`)        |
| Pods                | No                     | Yes                                     |
| systemd integration | Indirect               | Native (Quadlet)                        |
| Socket API          | `/var/run/docker.sock` | Optional (`podman.sock`)                |
| CLI commands        | `docker`               | `podman` (or use `podman-docker` alias) |

---

## 🧭 Mental Model

Think of **Docker** as “one big boss” (the daemon) that manages your containers for you.
Think of **Podman** as “direct control” — you manage containers like regular Linux processes, optionally grouped into pods and controlled by systemd.

You lose the babysitter daemon, but you gain:

* Simpler architecture
* Rootless safety
* Native system integration
* Kubernetes-aligned concepts (pods, Quadlet)

---

## 🧩 Example Setup (MySQL + Backend)

`docker-compose.yml` still works unchanged:

```yaml
services:
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: appdb
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
  backend:
    image: backend:latest
    depends_on:
      - mysql
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/appdb
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
    ports:
      - "8080:8080"
volumes:
  mysql_data:
```

Run it:

```bash
podman-compose up -d
```

---

## ✅ Rule of Thumb

* Everything you know from Docker works here.
* The key differences:

  1. No daemon (each container is its own process).
  2. Rootless by default.
  3. Pods and Quadlet add system-level flexibility.
  4. Compose exists separately but behaves the same.

Podman is Docker’s spiritual successor — same workflow, just stripped of the daemon and built with Linux-native elegance.





