---
title: SQL Server
date: 2025-10-25
tags:
    - containers
    - compose
    - sql-server
    - database
summary: Reusable setup for running Microsoft SQL Server 2022 using a Compose file — works with Docker or Podman — plus a portable init job and DBeaver connection notes.
aliases:
    - SQL Server Compose

---

# 🧱 SQL Server with Compose (Docker or Podman)

This cheat sheet spins up **Microsoft SQL Server 2022** in a container with persistent storage,
adds a **portable init job** (runs `.sql` files once), and connects from **DBeaver**.
The YAML works with **Docker** or **Podman**.

---

## Engine setup

Alias one variable to make commands engine-agnostic:

```bash
# Optional global engine switch
ENGINE=${ENGINE:-docker}  # set ENGINE=podman if you use Podman
```

Then every command is the same:

```bash
$ENGINE compose up -d
$ENGINE compose logs -f mssql
$ENGINE exec -it mssql /bin/bash
```

---

## SQL Server Compose File

```yaml
# File: compose.yaml
# Purpose: Local SQL Server 2022 with a one-time init job. Works on Docker or Podman.

services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql
    restart: unless-stopped

    environment:
      ACCEPT_EULA: "Y"                # Mandatory
      MSSQL_SA_PASSWORD: "Str0ng!Passw0rd"  # Must meet SQL Server complexity rules
      MSSQL_PID: "Developer"          # Developer (free) | Express | Standard | Enterprise (licensed)
      TZ: "Europe/Vilnius"

    ports:
      - "1433:1433"                   # If busy, use "11433:1433" and adjust clients to port 11433

    volumes:
      - mssql_data:/var/opt/mssql           # Data files live here
      - ./backups:/var/opt/mssql/backups:Z  # Optional: host-mounted backups dir (SELinux :Z ok on Docker too)

    # Simple TCP healthcheck that doesn't rely on sqlcmd being present in this image
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'echo > /dev/tcp/127.0.0.1/1433'"]
      interval: 10s
      timeout: 3s
      retries: 30
      start_period: 30s

  # One-time init job: runs *.sql in ./initdb against the mssql service after it's healthy
  mssql-init:
    image: mcr.microsoft.com/mssql-tools:latest
    container_name: mssql-init
    depends_on:
      mssql:
        condition: service_healthy
    environment:
      MSSQL_SA_PASSWORD: "Str0ng!Passw0rd"
    volumes:
      - ./initdb:/initdb:ro,Z
    entrypoint: ["/bin/bash", "-lc"]
    command: >
      '
      set -euo pipefail;
      HOST=mssql; PORT=1433; USER=sa; PASS="$MSSQL_SA_PASSWORD";
      # wait for TCP just in case
      for i in {1..60}; do
        (echo > /dev/tcp/${HOST}/${PORT}) >/dev/null 2>&1 && break || sleep 2;
      done;
      # run all .sql files in lexical order; ignore if directory empty
      shopt -s nullglob;
      for f in /initdb/*.sql; do
        echo \"[mssql-init] Running: $f\";
        /opt/mssql-tools/bin/sqlcmd -S ${HOST},${PORT} -U ${USER} -P \"${PASS}\" -i \"$f\";
      done;
      echo \"[mssql-init] Done.\"
      '

    # Run once on first `up`; remove after success
    restart: "no"

volumes:
  mssql_data:
```

**Notes**

* The **official SQL Server image does not auto-run** `docker-entrypoint-initdb.d` like MySQL/Postgres.
  The `mssql-init` job above is the portable way to run your own `.sql` on first boot (or whenever you want).
* On SELinux systems (Fedora/RHEL), the `:Z` suffix safely relabels host mounts. It’s harmless elsewhere.
* For **Podman rootless**, published ports are on the user namespace (localhost) — perfect for dev.

---

## 🧭 Run & Inspect

```bash
$ENGINE compose up -d
$ENGINE compose logs -f mssql      # wait for: "SQL Server is now ready for client connections"
$ENGINE compose logs -f mssql-init # see init job output (it exits when done)
$ENGINE ps
```

---

## 🧩 Connect (e.g., DBeaver)

* **Driver**: SQL Server (Microsoft)
* **Host**: `localhost`
* **Port**: `1433` (or your remapped host port)
* **Database**: e.g., `AppDb` (if you create one)
* **User/Pass**: `sa` / `Str0ng!Passw0rd`

If the driver requires, set properties such as:

* `encrypt=true;trustServerCertificate=true` for local dev,
* or import a proper certificate and set `trustServerCertificate=false`.

---

## 📁 Init scripts (`./initdb`)

All `.sql` files in `./initdb` are run by the **mssql-tools** job, in lexical order.

```
./initdb/
 ├─ 01-database-and-login.sql
 ├─ 02-schema.sql
 └─ 03-seed.sql
```

### Example — Create DB, login, user (`01-database-and-login.sql`)

```sql
-- Create database
IF DB_ID(N'AppDb') IS NULL
  CREATE DATABASE AppDb;
GO

-- Create server login
IF SUSER_ID(N'app_login') IS NULL
  CREATE LOGIN app_login WITH PASSWORD = 'AppUser!234';
GO

-- Map a database user to the login
USE AppDb;
GO
IF USER_ID(N'app_user') IS NULL
  CREATE USER app_user FOR LOGIN app_login;
GO

-- Grant a sane role; tighten later as needed
ALTER ROLE db_owner ADD MEMBER app_user;
GO
```

### Example — Schema (`02-schema.sql`)

```sql
USE AppDb;
GO

IF OBJECT_ID(N'dbo.customers', N'U') IS NULL
BEGIN
  CREATE TABLE dbo.customers(
    id INT IDENTITY(1,1) PRIMARY KEY,
    email NVARCHAR(255) NOT NULL UNIQUE,
    first_name NVARCHAR(100) NOT NULL,
    last_name NVARCHAR(100) NOT NULL,
    status NVARCHAR(20) NOT NULL DEFAULT N'ACTIVE',
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME()
  );
END
GO

CREATE INDEX IX_customers_email ON dbo.customers(email);
GO
```

### Example — Seed (`03-seed.sql`)

```sql
USE AppDb;
GO

MERGE dbo.customers AS tgt
USING (VALUES
  (N'ada@example.com',  N'Ada',   N'Lovelace', N'ACTIVE'),
  (N'grace@example.com',N'Grace', N'Hopper',   N'ACTIVE')
) AS src(email, first_name, last_name, status)
ON tgt.email = src.email
WHEN MATCHED THEN
  UPDATE SET first_name = src.first_name, last_name = src.last_name, status = src.status
WHEN NOT MATCHED THEN
  INSERT(email, first_name, last_name, status)
  VALUES(src.email, src.first_name, src.last_name, src.status);
GO
```

---

## 🔑 Create additional logins/users (interactive)

```bash
# Start an interactive sqlcmd without installing tools on the host
$ENGINE run --rm -it --network host mcr.microsoft.com/mssql-tools:latest \
  /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'Str0ng!Passw0rd'
```

```sql
CREATE LOGIN dev_login WITH PASSWORD = 'Dev!Passw0rd';
GO
USE AppDb;
GO
CREATE USER dev_user FOR LOGIN dev_login;
GO
ALTER ROLE db_datareader ADD MEMBER dev_user;  -- read-only example
GO
QUIT
```

> On Docker for Linux, `--network host` works. On Podman, it also works in rootless mode.
> If you’d rather use the Compose network, replace `--network host` with `--network <yourstack_default>` and `-S mssql`.

---

## ♻️ Re-run init scripts

Want to rerun everything from `./initdb`?

```bash
$ENGINE compose run --rm mssql-init
```

Want a **fresh** database?

```bash
$ENGINE compose down -v     # WARNING: also deletes the mssql_data volume (all data)
$ENGINE compose up -d
```

---

## 💾 Backups (inside container)

Mount `./backups` (already in YAML) and run:

```bash
# open a shell in the server container
$ENGINE exec -it mssql /bin/bash

# then:
# Full backup to the mounted host directory
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'Str0ng!Passw0rd' -Q \
  "BACKUP DATABASE AppDb TO DISK = N'/var/opt/mssql/backups/AppDb_full.bak' WITH INIT, COMPRESSION, STATS = 10;"
```

Restore example (target DB must be unused):

```sql
-- inside sqlcmd
ALTER DATABASE AppDb SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
RESTORE DATABASE AppDb FROM DISK = N'/var/opt/mssql/backups/AppDb_full.bak' WITH REPLACE, STATS = 10;
ALTER DATABASE AppDb SET MULTI_USER;
GO
```

---

## ⚙️ Spring Boot JDBC quick ref

```properties
spring.datasource.url=jdbc:sqlserver://localhost:1433;databaseName=AppDb;encrypt=true;trustServerCertificate=true
spring.datasource.username=app_login
spring.datasource.password=AppUser!234
spring.jpa.hibernate.ddl-auto=validate
```

---

## 🧱 Admin & troubleshooting

```bash
$ENGINE compose logs -f mssql
$ENGINE exec -it mssql /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'Str0ng!Passw0rd' -Q "SELECT @@VERSION;"
$ENGINE exec -it mssql /bin/bash
$ENGINE compose down
$ENGINE compose down -v   # DANGER: wipes data
```

**Common pitfalls**

* **Password rejected**: must include upper, lower, digit, and symbol; length ≥ 8–12 (use strong values).
* **Port clash**: remap to `"11433:1433"` and point clients to port **11433**.
* **SSL nags in JDBC**: use `encrypt=true;trustServerCertificate=true` locally, and proper certs for shared envs.
* **SELinux denials**: keep `:Z` on host mounts or switch to named volumes only.

