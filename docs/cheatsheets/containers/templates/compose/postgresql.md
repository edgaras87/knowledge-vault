---
title: PostgreSQL
date: 2025-10-17
tags:
  - containers
  - compose
  - postgresql
  - database
summary: Reusable setup for running PostgreSQL 16 with a Compose file — works with Docker or Podman.
aliases:
  - PostgreSQL Compose
---

# 🐘 PostgreSQL with Compose (Docker or Podman)

Run a reusable local **PostgreSQL** container using a Compose file.
Works with **Docker** or **Podman** — pick your engine and move on.

---

## Engine setup

```bash
# Optional global engine switch
ENGINE=${ENGINE:-docker}   # set ENGINE=podman if you use Podman
````

Then every command is engine-agnostic:

```bash
$ENGINE compose up -d
$ENGINE compose logs -f pg
$ENGINE exec -it pg psql -U appuser -d appdb
```

---

## PostgreSQL Compose File

```yaml
# File: compose.yaml
# Purpose: Run a reusable local PostgreSQL 16 container that your Spring Boot app
# (running on host) and DBeaver/psql can connect to via localhost.

services:
  postgres:
    image: postgres:16
    container_name: postgres
    restart: unless-stopped

    environment:
      # ---- Initial credentials (applied only on first run; persisted in volume) ----
      # !! Change these in real use or move to a .env file.
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
      POSTGRES_DB: appdb

      # Optional but nice for consistent timestamps
      #TZ: Europe/Vilnius

      # Optional: control initdb encoding/locale for new cluster
      #POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --locale=C.UTF-8"

    ports:
      - "5432:5432"
      # If host 5432 is busy, remap: "5433:5432" (then JDBC port=5433)

    # Postgres server settings (put comments on their own lines)
    #command:
      # Examples; tweak as needed:
      #- -c
      #- shared_buffers=256MB
      #- -c
      #- max_connections=100
      #- -c
      #- timezone=Europe/Vilnius

    volumes:
      - pg_data:/var/lib/postgresql/data
      # Named volume for persistent data (recommended default on Linux).

      #- ./initdb:/docker-entrypoint-initdb.d:ro,Z
      # Optional: .sql/.sh here run once on first boot (when volume is empty).
      # Add :Z on SELinux systems; harmless elsewhere.

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB} -h localhost"]
      interval: 10s
      timeout: 5s
      retries: 10

  # Optional UI
  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pg_data:
```

---

## 🧭 Run & Inspect

```bash
$ENGINE compose up -d
$ENGINE compose logs -f pg     # watch for: "database system is ready to accept connections"
$ENGINE ps
```

---

## 🧩 Connect (DBeaver or psql)

* Host: `localhost` (container_name: `postgres` for Podman rootless pgadmin)
* Port: `5432` (or your remap, e.g., `5433`)
* DB: `appdb`
* User/Pass: `appuser` / `apppass`

psql examples:

```bash
$ENGINE exec -it pg psql -U appuser -d appdb
$ENGINE exec -it pg psql -U appuser -d appdb -c '\dt'
```

---

## 🔑 Create a new DB role/user (later)

```bash
$ENGINE exec -it pg psql -U appuser -d appdb
```

```sql
-- Create role with login and password
CREATE ROLE reporter LOGIN PASSWORD 'reporterpass';

-- Least-privilege: read-only on appdb
GRANT CONNECT ON DATABASE appdb TO reporter;
GRANT USAGE ON SCHEMA public TO reporter;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporter;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO reporter;
```

---

## 📁 Init scripts (`./initdb`)

Any `.sql` or executable `.sh` files in `./initdb` run **once** on the **first** container start
(when `pg_data` is empty). Files run alphabetically; prefix with numbers to enforce order:

```
./initdb/
 ├─ 01-schema.sql
 ├─ 02-seed.sql
 └─ 03-roles.sql
```

### 1) Example schema (`01-schema.sql`)

```sql
-- Ensure we're on the right DB (the entrypoint creates appdb for you)
-- \c appdb  -- not needed when run by entrypoint, but harmless

CREATE TABLE IF NOT EXISTS customers (
  id          BIGSERIAL PRIMARY KEY,
  email       VARCHAR(255) NOT NULL UNIQUE,
  first_name  VARCHAR(100) NOT NULL,
  last_name   VARCHAR(100) NOT NULL,
  status      TEXT NOT NULL DEFAULT 'ACTIVE',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS orders (
  id           BIGSERIAL PRIMARY KEY,
  customer_id  BIGINT NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  total_cents  INTEGER NOT NULL,
  currency     CHAR(3) NOT NULL DEFAULT 'EUR',
  placed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Helpful indexes (Postgres supports IF NOT EXISTS)
CREATE INDEX IF NOT EXISTS idx_orders_customer ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_customers_email ON customers(email);
```

### 2) Example seed data (`02-seed.sql`)

```sql
INSERT INTO customers (email, first_name, last_name, status)
VALUES
  ('ada@example.com',   'Ada',   'Lovelace', 'ACTIVE'),
  ('grace@example.com', 'Grace', 'Hopper',   'ACTIVE')
ON CONFLICT (email) DO UPDATE
  SET first_name = EXCLUDED.first_name,
      last_name  = EXCLUDED.last_name,
      status     = EXCLUDED.status;

-- Guarded inserts for orders
INSERT INTO orders (customer_id, total_cents, currency)
SELECT id, 1999, 'EUR'
FROM customers
WHERE email = 'ada@example.com'
  AND NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = customers.id AND o.total_cents = 1999
  );

INSERT INTO orders (customer_id, total_cents, currency)
SELECT id, 3499, 'EUR'
FROM customers
WHERE email = 'grace@example.com'
  AND NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = customers.id AND o.total_cents = 3499
  );
```

### 3) Optional roles/privileges (`03-roles.sql`)

```sql
-- Example: analytics user with read-only access
DO $$
BEGIN
  IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'reporter') THEN
    CREATE ROLE reporter LOGIN PASSWORD 'reporterpass';
  END IF;
END$$;

GRANT CONNECT ON DATABASE appdb TO reporter;
GRANT USAGE ON SCHEMA public TO reporter;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporter;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO reporter;
```

### Re-running init scripts

They only run when the data volume is **fresh**. To reset:

```bash
$ENGINE compose down -v
$ENGINE compose up -d
```

To avoid resets, make seeds **idempotent** with `ON CONFLICT`, `IF NOT EXISTS`, and guarded inserts.

---

## 🧱 Migrations (recommended later)

When schema evolves, use **Flyway** or **Liquibase** with Spring Boot. Versioned scripts like
`V1__init.sql`, `V2__add_indexes.sql`, … are safer and traceable.

---

## ⚙️ Spring Boot quick reminder (PostgreSQL)

`application.properties` essentials:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/appdb
spring.datasource.username=appuser
spring.datasource.password=apppass

# Choose one:
spring.jpa.hibernate.ddl-auto=update
# spring.jpa.hibernate.ddl-auto=validate
# spring.jpa.hibernate.ddl-auto=create
# spring.jpa.hibernate.ddl-auto=create-drop

#spring.jpa.hibernate.show-sql: false

# Optional: be explicit about dialect
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

`application.yaml` essentials:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/appdb
    username: appuser
    password: apppass
  jpa:
    hibernate:
      ddl-auto: validate # or validate | update | create | create-drop as needed
    #show-sql: false
```

> Using `./initdb` or Flyway/Liquibase? Prefer `ddl-auto=validate` to catch mismatches without altering the DB.

---

## 🪶 Notes on Podman vs Docker

* **YAML is identical.**
* **SELinux distros (Fedora/RHEL/etc.):** keep `:Z` on bind mounts like `./initdb:...:Z`.
* **Rootless mode:** publishes to `localhost` by default; great for dev.
* **Systemd integration (Podman):**
  `podman generate systemd --new --files pg`

Everything else “just works.”
