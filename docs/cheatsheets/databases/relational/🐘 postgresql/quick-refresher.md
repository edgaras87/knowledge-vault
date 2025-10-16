---
title: quick-refresher
tags: [ postgres,postgresql,sql,database,psql,docker,docker-compose,pgadmin,spring-boot,jdbc,ci-cd,performance,security,devops]
summary: Hands-on PostgreSQL cheatsheet covering setup, SQL basics, Docker/Compose, IDE integration, Spring Boot config, CI/CD usage, and production tuning (indexes, backups, pooling, security).     
---



# 🐘 Postgre-SQL: From Basics to Full-Stack Integration

PostgreSQL (often just **Postgres**) is a **powerful, open-source relational database** known for reliability, standards compliance, and extensibility.  
It’s the database behind everything from small apps to enterprise-scale systems.  
Think of it as the “developer’s Swiss army knife” for data — solid, flexible, and endlessly scriptable.

---

## ⚙️ 1. What PostgreSQL Actually Does

At its core, Postgres:
- Stores structured data in **tables** (rows and columns).
- Enforces **relations** via keys and constraints.
- Lets you query and transform data using **SQL**.
- Runs as a **daemon/service** on your system or inside a Docker container.
- Manages concurrent access safely with **MVCC** (multi-version concurrency control).

👉 In short: PostgreSQL is a **transactional data engine** — designed for consistency, safety, and complex queries.

---

## 🧱 2. Core Concepts

| Concept | Description |
|----------|-------------|
| **Database** | Logical container for schemas, tables, and users. |
| **Schema** | Namespace inside a database (like a folder for tables). |
| **Table** | Structured data collection (rows = records, columns = fields). |
| **Row / Column** | Individual data unit / attribute. |
| **Primary key** | Unique identifier per row. |
| **Foreign key** | Relationship link between tables. |
| **Transaction** | Atomic unit of work — all or nothing. |
| **MVCC** | Allows simultaneous reads/writes safely via versioning. |

---

## 🧩 3. PostgreSQL in Action

### Start / Stop (Linux)

```bash
sudo systemctl start postgresql
sudo systemctl status postgresql
sudo -u postgres psql
```

### Quick DB shell commands

```sql
\l           -- list databases
\c mydb      -- connect to database
\dt          -- list tables
\d tablename -- describe table structure
\q           -- quit
```

### Create a user and database

```sql
CREATE USER devuser WITH PASSWORD 'secret';
CREATE DATABASE devdb OWNER devuser;
GRANT ALL PRIVILEGES ON DATABASE devdb TO devuser;
```

---

## 🧰 4. SQL Basics Refresher

```sql
-- Create table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email TEXT UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert data
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');

-- Query data
SELECT * FROM users;
SELECT name FROM users WHERE email LIKE '%@example.com';

-- Update
UPDATE users SET name='Alicia' WHERE id=1;

-- Delete
DELETE FROM users WHERE id=1;
```

---

## 🧩 5. PostgreSQL with Docker

Simplify local setup — run everything isolated and disposable.

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_USER=devuser \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=devdb \
  -p 5432:5432 \
  -v pg_data:/var/lib/postgresql/data \
  postgres:16
```

Now you can connect from your host or app via:

```
Host: localhost
Port: 5432
Database: devdb
User: devuser
Password: secret
```

### Docker Compose Example

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - postgres

volumes:
  pg_data:
```

Access **pgAdmin** at [http://localhost:5050](http://localhost:5050) → connect to host `postgres`, port `5432`.

---

## 🧠 6. Common Data Types

| Type                | Example        | Notes                             |
| ------------------- | -------------- | --------------------------------- |
| `INTEGER`           | 42             | Whole numbers                     |
| `SERIAL`            | auto-increment | Shortcut for `integer + sequence` |
| `VARCHAR(n)`        | 'Hello'        | Variable-length string            |
| `TEXT`              | Long article   | Unlimited string                  |
| `BOOLEAN`           | TRUE / FALSE   | Logical flag                      |
| `DATE`, `TIMESTAMP` | 2025-10-15     | Temporal data                     |
| `JSONB`             | '{"x":1}'      | Binary JSON — queryable!          |
| `ARRAY`             | '{1,2,3}'      | PostgreSQL supports array columns |

---

## 🧩 7. User & Access Management

```sql
-- Create user
CREATE USER api_user WITH PASSWORD 'apipass';

-- Grant permissions
GRANT CONNECT ON DATABASE devdb TO api_user;
GRANT USAGE ON SCHEMA public TO api_user;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO api_user;

-- Future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE ON TABLES TO api_user;
```

👉 Keep app-level users **least-privileged** — don’t give them superuser rights.

---

## 🧩 8. Useful CLI Commands

```bash
psql -U devuser -d devdb        # connect to DB
psql -h localhost -U devuser    # specify host
pg_dump -U devuser devdb > dump.sql    # backup
psql -U devuser -d devdb < dump.sql    # restore
```

Check service status:

```bash
sudo systemctl status postgresql
```

---

## 🧰 9. Troubleshooting

| Issue                         | Fix                                                  |
| ----------------------------- | ---------------------------------------------------- |
| Can't connect (local)         | Check `pg_hba.conf` and open port 5432.              |
| “role does not exist”         | Create user via `CREATE USER`.                       |
| Permission denied             | Ensure grants on database and schema.                |
| Docker container forgets data | Add a volume: `-v pg_data:/var/lib/postgresql/data`. |
| Encoding issues               | Use `UTF8` during DB creation.                       |

---

## 🧩 10. PostgreSQL Configuration Essentials

Main config files (Linux or container):

```
/etc/postgresql/16/main/postgresql.conf
/etc/postgresql/16/main/pg_hba.conf
```

Common tweaks:

```bash
listen_addresses = '*'
max_connections = 100
shared_buffers = 256MB
work_mem = 16MB
logging_collector = on
log_directory = 'log'
```

Reload without restart:

```bash
sudo systemctl reload postgresql
```

---

## 💻 11. PostgreSQL in IDEs (JetBrains / VS Code)

### JetBrains (DataGrip, IntelliJ Ultimate)

* Open “Database” panel → “+” → “PostgreSQL”.
* Set host/port/user/password.
* Can run SQL scripts directly, browse schema, or diff databases.
* Integration with **.env** variables and **Docker Compose services**.

**Shortcut magic:**

* Alt+Enter → run query under cursor
* Ctrl+Enter → run entire script
* Ctrl+Shift+F10 → execute file

---

### VS Code Setup

Extensions to install:

* **SQLTools**
* **SQLTools PostgreSQL Driver**
* **vscode-database-client** (optional GUI)

Example `.sqltools.json`:

```json
{
  "connections": [
    {
      "name": "Local Postgres",
      "driver": "PostgreSQL",
      "previewLimit": 50,
      "server": "localhost",
      "port": 5432,
      "database": "devdb",
      "username": "devuser",
      "password": "secret"
    }
  ]
}
```

Run queries directly in VS Code or integrated terminal.

---

## 🚀 12. Spring Boot Integration Example

In your `application.properties`:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/devdb
spring.datasource.username=devuser
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

Docker Compose-friendly URL:

```
jdbc:postgresql://postgres:5432/devdb
```

---

## 🧠 13. Developer Workflow Pattern

1. Start your DB (local or Docker).
2. Connect through IDE or pgAdmin.
3. Run schema migrations (Liquibase/Flyway).
4. Run app → verify connection.
5. Backup before experimenting.

Keep `.env` variables for credentials:

```
POSTGRES_USER=devuser
POSTGRES_PASSWORD=secret
POSTGRES_DB=devdb
POSTGRES_PORT=5432
```

Load via:

```bash
export $(grep -v '^#' .env | xargs)
```

---

## 🧩 14. Optimization and Maintenance

* **Analyze & Vacuum**

  ```sql
  VACUUM ANALYZE;
  ```

  Keeps statistics fresh and space reclaimed.

* **Indexes**

  ```sql
  CREATE INDEX idx_users_email ON users(email);
  ```

* **Backups**

  ```bash
  pg_dumpall > full_backup.sql
  ```

* **Performance**

  * Prefer `JSONB` for flexible data.
  * Avoid `SELECT *` in production queries.
  * Tune `work_mem`, `shared_buffers` for heavy loads.

---

## ✅ 15. Summary

* PostgreSQL = reliable, standard, and developer-friendly.
* Use Docker for easy, isolated environments.
* Use pgAdmin or IDE integration for management.
* Integrate cleanly with Spring Boot via JDBC.
* Keep permissions minimal, backups regular, and queries explicit.

---


## 💻 16. PostgreSQL in Developer Workflows (JetBrains & VS Code)

Postgres is more than a database service — it’s part of your **daily dev feedback loop**.  
Connecting it tightly to your IDE and environment lets you debug, test, and deploy confidently.

---

### 🧩 JetBrains IDEs (IntelliJ Ultimate / DataGrip / PyCharm Pro)

| Action | Description |
|--------|--------------|
| **Add Data Source → PostgreSQL** | Connect via host/port or directly through Docker Compose service. |
| **Run SQL scripts inline** | Execute queries from `.sql` files, see live results. |
| **Compare Schemas** | Diff local vs. remote DB structures (great for migrations). |
| **Generate DDL from tables** | Auto-create SQL definitions for export or review. |
| **Inspect Query Plan (Ctrl+Shift+Enter)** | Visualize query performance via `EXPLAIN ANALYZE`. |

**Quick habit loop:**
1. Write query → Alt+Enter → Run.  
2. Fix slow query → Inspect execution plan.  
3. Save snippet in IDE’s “Scratch File” for reuse.

You can connect directly to your Dockerized DB by pointing to host `localhost:5432` or service name `postgres` inside Compose.

---

### 🧠 VS Code Integration

**Recommended extensions:**
- **SQLTools** + **SQLTools PostgreSQL Driver** → simple queries, table explorer.  
- **Database Client** → visual schema browsing.  
- **.env support** → use environment variables for credentials.  

Connect via `.sqltools.json`:
```json
{
  "connections": [
    {
      "name": "Local Postgres",
      "driver": "PostgreSQL",
      "server": "localhost",
      "port": 5432,
      "database": "devdb",
      "username": "devuser",
      "password": "secret"
    }
  ]
}
````

You can run and test queries directly in `.sql` files without switching tools.

---

## 🧰 17. PostgreSQL + Docker Compose for Multi-Environments

A single Compose file can handle both **development** and **testing** databases.

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d devdb"]
      interval: 10s
      retries: 5

  testdb:
    image: postgres:16
    environment:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: testdb
    ports:
      - "55432:5432"
    tmpfs:
      - /var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      retries: 3

volumes:
  pg_data:
```

**Why it matters:**

* `pg_data` → persists your dev data.
* `testdb` → ephemeral; wiped between test runs.
* Healthchecks prevent dependent containers from starting too early.

👉 You can link your backend app to `postgres` (dev) or `testdb` (CI) just by switching env variables.

---

## 🚀 18. Integration with CI/CD (GitHub Actions Example)

```yaml
name: Backend Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports: ['5432:5432']
        options: >-
          --health-cmd="pg_isready -U testuser -d testdb"
          --health-interval=5s --health-timeout=5s --health-retries=5
    steps:
      - uses: actions/checkout@v4
      - name: Wait for Postgres
        run: |
          until pg_isready -h localhost -U testuser; do
            sleep 1
          done
      - name: Run Tests
        run: ./gradlew test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass
```

This pattern runs Postgres automatically for tests — no manual setup, no external dependencies.

---

## 🧩 19. Managing Environments: Local → Staging → Production

**Environment separation rule:**

* Keep configuration externalized (`.env`, `configmaps`, or Secrets).
* Never commit real passwords or keys.
* Distinguish database URLs by environment:

| Environment | Example URL                                        | Notes                   |
| ----------- | -------------------------------------------------- | ----------------------- |
| Local       | `jdbc:postgresql://localhost:5432/devdb`           | Docker or local install |
| Staging     | `jdbc:postgresql://staging-db.internal:5432/appdb` | Shared test data        |
| Production  | `jdbc:postgresql://prod-db.cluster:5432/appdb`     | High availability       |

In Docker Compose:

```yaml
env_file:
  - .env.${ENVIRONMENT}
```

In Spring Boot:

```properties
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASS}
```

**Pro tip:** version `.env.example` for onboarding new developers easily.

---

## 🔒 20. Production Hardening & Maintenance Patterns

### Connection Pooling

Postgres performs best with connection pooling (e.g., **PgBouncer**).

```bash
docker run -d \
  --name pgbouncer \
  -p 6432:6432 \
  -e DB_USER=devuser \
  -e DB_PASSWORD=secret \
  -e DB_HOST=postgres \
  edoburu/pgbouncer
```

App connects to `localhost:6432` instead of `5432`.

### Backups

```bash
pg_dump -U devuser devdb | gzip > backup_$(date +%F).sql.gz
```

Automate via cron or CI.

### Replication and Scaling

* Use **Streaming Replication** for read replicas.
* Or scale horizontally via **logical replication**.
* Consider **Patroni** or **TimescaleDB** for high-availability setups.

### Security

* Disable remote superuser login.
* Restrict `listen_addresses` in `postgresql.conf`.
* Rotate credentials regularly.
* Encrypt connections with SSL (`ssl = on`).

---

## ✅ Summary (Developer Edition)

* Integrate Postgres into your IDE and CI/CD pipelines, not just your runtime.
* Run it via Docker Compose to match local and production schemas.
* Automate validation, migrations, and backups.
* Use `.env` files to keep environments cleanly separated.
* Monitor performance (`EXPLAIN ANALYZE`, logs, connection stats) like a pro DBA.

---
