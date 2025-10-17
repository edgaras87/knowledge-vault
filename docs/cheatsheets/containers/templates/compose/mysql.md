---
title: MySQL
date: 2025-10-17
tags:
  - containers
  - compose
  - mysql
  - database
summary: Reusable setup for running MySQL 8 using a Compose file — works with Docker or Podman.
aliases:
  - MySQL Compose
---

# 🐬 MySQL with Compose (Docker or Podman)

This cheat sheet shows how to run a reusable local **MySQL 8** container using a Compose file.
It works the same with **Docker** or **Podman** — choose your engine and forget about the rest.

---

## Engine setup

You can alias one variable to simplify commands:

```bash
# Optional global engine switch
ENGINE=${ENGINE:-docker}  # set ENGINE=podman if you use Podman
````

Then every command becomes engine-agnostic:

```bash
$ENGINE compose up -d
$ENGINE compose logs -f mysql
$ENGINE exec -it mysql8 mysql -u root -p
```

---

## MySQL Compose File

```yaml
# File: compose.yaml
# Purpose: Run a reusable local MySQL 8 container that your Spring Boot app
# (running on host) and DBeaver can connect to via localhost. Works with Docker or Podman.
services:
  mysql:
    image: mysql:8.0              # Base image. For reproducibility, pin a patch: e.g., mysql:8.0.43
    container_name: mysql8        # Stable handle for exec/logs: `$ENGINE exec -it mysql8 ...`
    restart: unless-stopped       # Auto-start on reboot; won't restart if you stop it manually

    environment:
      MYSQL_ROOT_PASSWORD: rootpass  # Root password (only used on first init). Move to .env for safety.
      MYSQL_DATABASE: appdb          # Auto-create this DB on first init
      MYSQL_USER: appuser            # App user created on first init
      MYSQL_PASSWORD: apppass        # App user password (first init only)
      TZ: Europe/Vilnius             # Align container time with host; helpful for logs/JDBC timestamps

    ports:
      - "3306:3306"               # Host:Container. If host 3306 is busy, use "3307:3306" and set JDBC port=3307

    command:                      # mysqld server flags
      - --default-authentication-plugin=mysql_native_password
        # Use legacy auth for maximum client compatibility (DBeaver/JDBC); drop this if all clients
        # support caching_sha2_password (the modern default).
      - --character-set-server=utf8mb4
        # Full Unicode, including emoji.
      - --collation-server=utf8mb4_0900_ai_ci
        # Modern, case-insensitive, accent-insensitive collation that pairs with utf8mb4.
      - --skip-host-cache
        # Avoid reverse DNS lookups on connect (faster, fewer surprises).
      - --skip-name-resolve
        # Force MySQL to use IPs instead of hostnames (prevents weird grants on hostnames).

    volumes:
      - mysql_data:/var/lib/mysql
        # Named volume for data files (safe default on Linux; survives container recreation).
      - ./initdb:/docker-entrypoint-initdb.d:ro,Z
        # One-time init scripts (.sql/.sh) executed alphabetically on first boot (empty data dir).
        # `:ro` = read-only for safety. `,Z` = SELinux relabel (Fedora/RHEL); harmless elsewhere.

    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -p$${MYSQL_ROOT_PASSWORD} --silent"]
        # Simple readiness probe. Note the $$ to escape $ in YAML so the container sees $MYSQL_ROOT_PASSWORD.
      interval: 10s               # Probe every 10 seconds
      timeout: 5s                 # Consider it failed if no response within 5 seconds
      retries: 10                 # Require 10 consecutive passes/failures before flipping status

volumes:
  mysql_data:                     # Declare the named volume used above
```

---

## 🧭 Run & Inspect

```bash
$ENGINE compose up -d
$ENGINE compose logs -f mysql   # watch for: "ready for connections"
$ENGINE ps
```

---

## 🧩 Connect (e.g., DBeaver)

* Host: `localhost`
* Port: `3306` (or remapped port)
* DB: `appdb`
* User/Pass: `appuser` / `apppass` (standard)
  or `root` / `rootpass` (admin)

---

## 🔑 Create new DB users

```bash
$ENGINE exec -it mysql8 mysql -u root -p
# enter rootpass
```

```sql
CREATE USER 'newuser'@'%' IDENTIFIED BY 'newpass';
GRANT ALL PRIVILEGES ON appdb.* TO 'newuser'@'%';
FLUSH PRIVILEGES;
```

---

## 📁 Init scripts (`./initdb`)

Any `.sql` or `.sh` files placed in `./initdb` run **once** on the **first boot** (when the volume is empty).

```
./initdb/
 ├─ 01-schema.sql
 ├─ 02-seed.sql
 └─ 03-users.sql
```

### Example — Schema (`01-schema.sql`)

```sql
USE appdb;

CREATE TABLE IF NOT EXISTS customers (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  status ENUM('ACTIVE','INACTIVE') NOT NULL DEFAULT 'ACTIVE',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

---

### Example — Seed Data (`02-seed.sql`)

```sql
USE appdb;

INSERT INTO customers (email, first_name, last_name, status)
VALUES
  ('ada@example.com',  'Ada',  'Lovelace', 'ACTIVE'),
  ('grace@example.com','Grace','Hopper',   'ACTIVE')
ON DUPLICATE KEY UPDATE
  first_name = VALUES(first_name),
  last_name  = VALUES(last_name),
  status     = VALUES(status);
```

---

## ♻️ Re-run init scripts

If you want a clean start:

```bash
$ENGINE compose down -v
$ENGINE compose up -d
```

Otherwise, make your SQL idempotent using:

* `CREATE TABLE IF NOT EXISTS`
* `INSERT ... ON DUPLICATE KEY UPDATE`
* `CREATE INDEX IF NOT EXISTS`

---

## 🧱 Migrations (recommended later)

For production or evolving schemas, manage changes with **Flyway** or **Liquibase** in Spring Boot instead of raw SQL.

---

## ⚙️ Spring Boot JPA Strategy Reminder

```properties
# pick ONE:
spring.jpa.hibernate.ddl-auto=update
# spring.jpa.hibernate.ddl-auto=validate
# spring.jpa.hibernate.ddl-auto=create
# spring.jpa.hibernate.ddl-auto=create-drop
```

> If you’re using `./initdb` or Flyway, set `ddl-auto=validate` to ensure schema consistency.

---

## 🪶 Notes on Podman vs Docker

* YAML is identical.
* `:Z` volume label may be required on SELinux (Fedora/RHEL).
* Rootless mode publishes ports to localhost only (fine for dev).
* You can generate a systemd unit with:
  `podman generate systemd --new --files mysql8`

Everything else “just works.”

---

That’s it — a single Compose file that runs anywhere: Docker, Podman, WSL, or a full Linux host.

If you later add PostgreSQL or Redis, you can reuse this same `$ENGINE compose` pattern — one variable, no mental friction.


