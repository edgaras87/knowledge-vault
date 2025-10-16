---
title: fullstack-deployment-overview
tags: [architecture, deployment, fullstack, docker, nginx, postgresql, redis, systemd, cicd, monitoring]
summary: Comprehensive overview of deploying and operating a full stack application using Docker, Nginx, PostgreSQL, Redis, systemd, CI/CD, and monitoring tools.
---

# 🚀 Full Stack Deployment & Operations Overview

### (Frontend + Backend + Infrastructure + CI/CD)

This document connects everything — from developer commits to live, monitored systems.  
It shows how Git, Docker, Nginx, PostgreSQL, Redis, and systemd interact across environments, supported by CI/CD pipelines and monitoring.

---

## 🧩 1. The Complete Stack

```
            ┌────────────────────────────┐
            │         Developer          │
            │   (Git + IDE + Docker)     │
            └────────────┬───────────────┘
                         │ push/build
                         ▼
     ┌──────────────────────────────┐
     │ Continuous Integration (CI) │
     │   Build → Test → Package    │
     └────────────┬────────────────┘
                  │
                  ▼
     ┌──────────────────────────────┐
     │ Continuous Deployment (CD)  │
     │   Deploy → Start Services    │
     └────────────┬────────────────┘
                  │
                  ▼
     ┌──────────────────────────────┐
     │     Production Server       │
     │  (systemd + Docker + Nginx) │
     └────────────┬────────────────┘
                  │
                  ▼
     ┌──────────────────────────────┐
     │     Monitoring & Logs        │
     │  (Prometheus, Grafana, ELK)  │
     └──────────────────────────────┘
```



Everything begins with **Git**, moves through **CI/CD automation**, lands on a **Dockerized host** managed by **systemd**, and is served to the world through **Nginx**.

---

## ⚙️ 2. Environments and Their Roles

| Environment | Purpose | Key Tools |
|--------------|----------|-----------|
| **Local** | Fast iteration, testing | Docker Compose, local DB |
| **Staging** | Full stack replica | Docker Compose, CI/CD |
| **Production** | Stable live system | Docker, systemd, Nginx |
| **CI Runner** | Automated testing | GitHub Actions / GitLab CI |

**Golden rule:**  
> Each environment should be identical in architecture, differing only in configuration.

---

## 🧱 3. Stack Layers Overview

| Layer | Component | Purpose |
|--------|------------|----------|
| **Source Control** | Git | Version all code and infrastructure |
| **Build Layer** | Node.js, Gradle/Maven | Build frontend + backend artifacts |
| **Runtime Layer** | Docker | Run isolated containers |
| **Routing Layer** | Nginx | Route external traffic |
| **Data Layer** | PostgreSQL, Redis | Persistent + cached data |
| **Orchestration Layer** | systemd | Ensure uptime and startup order |
| **Automation Layer** | CI/CD | Test, build, and deploy automatically |
| **Observation Layer** | Prometheus, Grafana, Logs | Metrics, alerts, traces |

---

## 🐳 4. Docker Compose for Unified Stack

The glue that connects your local and staging environments.

```yaml
version: "3.9"
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on: [backend, frontend]

  frontend:
    build: ./frontend
    expose:
      - "5173"

  backend:
    build: ./backend
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/appdb
      SPRING_DATASOURCE_USERNAME: devuser
      SPRING_DATASOURCE_PASSWORD: secret
    depends_on: [postgres, redis]

  postgres:
    image: postgres:16
    volumes:
      - pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    volumes:
      - redis_data:/data

volumes:
  pg_data:
  redis_data:
```

This setup runs the **entire full stack** locally, exactly as it would in staging or production.

---

## 🔁 5. CI/CD Pipeline Flow

**Continuous Integration (CI)** ensures your build works and tests pass.
**Continuous Deployment (CD)** delivers it safely to your server.

### Example (GitHub Actions)

```yaml
name: Build & Deploy Full Stack
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build backend
        run: ./gradlew build
      - name: Build frontend
        run: npm ci && npm run build
      - name: Build Docker images
        run: |
          docker build -t ghcr.io/user/backend:${{ github.sha }} backend/
          docker build -t ghcr.io/user/frontend:${{ github.sha }} frontend/
      - name: Push Images
        run: |
          docker push ghcr.io/user/backend:${{ github.sha }}
          docker push ghcr.io/user/frontend:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: SSH & Deploy
        run: |
          ssh user@server "
            docker pull ghcr.io/user/backend:${{ github.sha }} &&
            docker pull ghcr.io/user/frontend:${{ github.sha }} &&
            docker compose up -d &&
            sudo systemctl reload nginx
          "
```

This pipeline:

1. Builds both backend and frontend.
2. Pushes images to a container registry.
3. SSHes into the server and redeploys.
4. Reloads Nginx to apply new frontend files.

---

## ⚡ 6. Deployment Directory Structure (on server)

```
/opt/app/
 ├─ docker-compose.yml
 ├─ nginx.conf
 ├─ .env
 ├─ frontend/
 ├─ backend/
 ├─ logs/
 └─ volumes/
     ├─ postgres/
     └─ redis/
```

`systemd` runs Docker as the service manager underneath:

```bash
sudo systemctl restart docker
sudo docker compose up -d
```

---

## 🧩 7. Nginx as the Traffic Controller

Handles requests, SSL, static serving, and proxying:

```nginx
server {
    listen 80;
    server_name example.com;
    location / {
        root /usr/share/nginx/html;
        try_files $uri /index.html;
    }
    location /api/ {
        proxy_pass http://backend:8080;
    }
}
```

Nginx routes browser traffic → frontend,
and `/api` calls → backend → PostgreSQL/Redis.

---

## 🐘 8. Databases and Persistence

### PostgreSQL

* Stores structured, durable data.
* Mounted via Docker volume (`pg_data`).
* Managed by `systemd` through Docker.

### Redis

* In-memory caching + sessions.
* Mounted volume for optional persistence (`redis_data`).
* Communicates over internal Docker network.

---

## 🧠 9. Configuration & Secrets Management

All environments read from `.env` files:

```
DB_USER=devuser
DB_PASS=secret
REDIS_PASS=redispass
API_KEY=some_key
```

For production:

* Use `.env.prod` with stronger creds.
* Never commit secrets — use CI/CD secrets storage.

---

## 💾 10. Backup & Recovery

### PostgreSQL backup:

```bash
pg_dump -U devuser appdb | gzip > backup_$(date +%F).sql.gz
```

### Redis snapshot:

```bash
redis-cli save
```

Automate backups with systemd timers or cron.

---

## 🧭 11. Monitoring & Logging

| Tool                                              | Purpose                |
| ------------------------------------------------- | ---------------------- |
| **journalctl**                                    | OS & service logs      |
| **Docker logs**                                   | Container-level events |
| **Prometheus**                                    | Metrics collection     |
| **Grafana**                                       | Visualization & alerts |
| **ELK stack (Elasticsearch + Logstash + Kibana)** | Centralized logging    |

Minimal local setup:

```bash
docker run -d -p 9090:9090 prom/prometheus
docker run -d -p 3000:3000 grafana/grafana
```

---

## 🔒 12. Security Layers

| Layer           | Defense                                      |
| --------------- | -------------------------------------------- |
| **Network**     | Nginx firewall rules, fail2ban               |
| **Transport**   | HTTPS via Let’s Encrypt                      |
| **Application** | Authentication, rate limiting                |
| **Data**        | Encrypted DB connections                     |
| **Access**      | SSH keys, non-root Docker users              |
| **Secrets**     | Environment variables in CI/CD secrets store |

---

## 🔁 13. Continuous Maintenance Workflow

| Task                | Tool                  | Frequency     |
| ------------------- | --------------------- | ------------- |
| Code updates        | Git                   | Continuous    |
| Build & deploy      | CI/CD                 | On every push |
| Logs review         | journalctl / Grafana  | Daily         |
| Backup rotation     | systemd timer         | Daily/weekly  |
| Security patches    | apt, Docker images    | Weekly        |
| Service healthcheck | systemctl, Prometheus | Ongoing       |

---

## 🧩 14. Disaster Recovery Pattern

1. Restore from latest DB + Redis backups.
2. Pull latest images from registry.
3. Deploy via `docker compose up -d`.
4. Reconnect DNS / certificates.
5. Verify via Nginx health endpoints.

**Recovery time objective:** minutes, not hours.

---

## 🧠 15. Developer-to-Production Mental Model

| Role           | Tool         | Responsibility               |
| -------------- | ------------ | ---------------------------- |
| **Developer**  | Git, Docker  | Build and test features      |
| **Integrator** | CI           | Verify builds                |
| **Deployer**   | CD           | Push working containers live |
| **Operator**   | systemd      | Keep services healthy        |
| **Observer**   | Grafana/Logs | Detect issues early          |

---

## ✅ 16. Summary

* **Git** → tracks source and triggers builds.
* **Docker** → standardizes runtime.
* **Nginx** → routes traffic to services.
* **PostgreSQL** → stores persistent state.
* **Redis** → accelerates performance.
* **systemd** → ensures everything starts and stays alive.
* **CI/CD** → automates the entire loop.
* **Monitoring tools** → give visibility and peace of mind.

Everything runs as a **modular, reproducible, observable system** — a fully self-contained full stack.
