---
title: Quick Start 
tags: [nginx, webserver, reverse-proxy, load-balancer, https, caching, security]
summary: Nginx basics, reverse proxy, load balancing, HTTPS setup, caching, security, and common configurations.
aliases:
  - Nginx Quick Start
---


# 🌐 Nginx: From Basics to High-Performance Web Serving

Nginx (“engine-x”) is a **high-performance web server, reverse proxy, and load balancer**.  
It’s lightweight, event-driven, and designed to handle thousands of concurrent connections with minimal resources.  
You’ll find it serving static sites, routing API traffic, proxying backend apps, or terminating SSL — often all at once.

---

## ⚙️ 1. What Nginx Actually Does

Nginx can play multiple roles depending on configuration:

| Role | Description |
|------|--------------|
| **Web server** | Serves static files directly (HTML, JS, CSS, images). |
| **Reverse proxy** | Forwards client requests to backend apps (e.g., Node, Spring Boot). |
| **Load balancer** | Distributes requests across multiple backend servers. |
| **TLS terminator** | Handles HTTPS encryption before passing traffic internally. |
| **Cache layer** | Stores responses to reduce backend load. |

👉 In short: Nginx sits **between the internet and your application**, managing how traffic flows.

---

## 🧱 2. Core Concepts

| Concept | Description |
|----------|--------------|
| **Worker processes** | Handle client connections. Nginx scales by using multiple workers efficiently. |
| **Directives** | Configuration commands that define behavior (e.g., `listen`, `server_name`). |
| **Context blocks** | Hierarchical sections: `main`, `http`, `server`, and `location`. |
| **Server block** | Defines a virtual host — domain, ports, SSL, routes. |
| **Location block** | Defines how to handle requests matching specific URIs. |

---

## 🧩 3. File Structure Overview

Typical Linux layout after install:

```

/etc/nginx/
├─ nginx.conf          # main config (includes others)
├─ conf.d/             # custom site configs (enabled by default)
├─ sites-available/    # optional (Debian/Ubuntu layout)
├─ sites-enabled/      # symlinks to active sites
├─ snippets/           # reusable config fragments
└─ logs/
├─ access.log
└─ error.log
```

Test and reload Nginx safely:

```bash
sudo nginx -t       # test syntax
sudo systemctl reload nginx
sudo systemctl status nginx
```

---

## 🧰 4. Basic HTTP Server Example

**Goal:** Serve static files from `/var/www/html` on port 80.

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/html;
    index index.html;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

👉 This is Nginx as a **pure web server** — no proxying, just serving files.

---

## 🔁 5. Reverse Proxy Setup

**Goal:** Forward traffic from Nginx → backend app (e.g., Spring Boot or Node).

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

✅ Best practice:

* Always pass client IP headers.
* Use `proxy_pass` **without trailing slash** unless you understand path rewriting.
* Protect upstreams (never expose raw app ports to the internet).

---

## 🔒 6. HTTPS (TLS) Configuration

Using **Let’s Encrypt** certificates (managed by `certbot`):

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Manual example:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

👉 Always redirect HTTP → HTTPS.

---

## ⚡ 7. Load Balancing Example

Round-robin across two backend servers:

```nginx
upstream backend_cluster {
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://backend_cluster;
    }
}
```

Variants:

* `ip_hash;` for sticky sessions
* `least_conn;` for even load

---

## 🧠 8. Useful Directives & Variables

| Directive          | Purpose                       |
| ------------------ | ----------------------------- |
| `root`             | Directory to serve files from |
| `index`            | Default file to serve         |
| `server_name`      | Hostname match for requests   |
| `error_page`       | Custom error responses        |
| `rewrite`          | URL rewriting                 |
| `try_files`        | Fallbacks for static serving  |
| `proxy_pass`       | Forward to backend            |
| `proxy_set_header` | Pass headers to backend       |

**Common variables:**

* `$remote_addr` → client IP
* `$host` → domain in request
* `$uri` → path part of request
* `$request_uri` → original request including query
* `$upstream_addr` → backend server used

---

## 🧩 9. Logging and Monitoring

Logs are your best debugging friend:

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

Sample log format:

```
127.0.0.1 - - [15/Oct/2025:12:34:56 +0000] "GET /index.html HTTP/1.1" 200 612
```

You can define custom formats:

```nginx
log_format main '$remote_addr - $host [$time_local] "$request" $status $body_bytes_sent';
access_log /var/log/nginx/access.log main;
```

---

## 🧩 10. Caching Static Assets

```nginx
location ~* \.(jpg|jpeg|png|gif|css|js|ico|woff2?)$ {
    expires 30d;
    add_header Cache-Control "public, no-transform";
}
```

👉 Offload repeated requests from your backend and improve browser performance.

---

## 🧱 11. Security Hardening

* Disable server version info:

  ```nginx
  server_tokens off;
  ```
* Limit request size:

  ```nginx
  client_max_body_size 10M;
  ```
* Prevent clickjacking & MIME sniffing:

  ```nginx
  add_header X-Frame-Options SAMEORIGIN;
  add_header X-Content-Type-Options nosniff;
  ```
* Use rate limiting (basic DDOS protection):

  ```nginx
  limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
  location /api/ {
      limit_req zone=api burst=20;
      proxy_pass http://backend;
  }
  ```

---

## 🧩 12. Example Full Setup (Static + API Proxy + HTTPS)

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/html;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## 🧩 13. Nginx in Development Environments

### JetBrains (IDEA / PyCharm)

* You can **run and debug local servers** via “Edit Configurations → Nginx”.
* Syntax highlighting is built-in; test configs directly with `nginx -t`.
* Use `Deployment` tools to sync `/etc/nginx/` or container configs to remote hosts.

### VS Code

Install:

* **Nginx Configuration Language** (syntax highlighting)
* **Nginx Snippets** (ready-to-use config templates)
* **Docker Extension** (if you’re running Nginx in containers)

Run via Docker:

```bash
docker run -d -p 8080:80 -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf nginx
```

---

## 🧰 14. Troubleshooting

| Problem                   | Fix                                                     |
| ------------------------- | ------------------------------------------------------- |
| Config change not applied | `sudo nginx -t && sudo systemctl reload nginx`          |
| “Bad Gateway (502)”       | Backend down or wrong `proxy_pass` target               |
| Permission denied         | Ensure Nginx user (`www-data`/`nginx`) can access files |
| Infinite redirect loop    | Check `proxy_pass` URLs and rewrite rules               |
| SSL errors                | Verify certificate paths & permissions                  |

---

## 🧠 15. Advanced Topics (for later)

* Reverse proxy caching (`proxy_cache_path`, `proxy_cache`).
* HTTP/2 and QUIC/HTTP3 enablement.
* Gzip and Brotli compression.
* Load balancing with health checks.
* Serving multiple domains (SNI).
* Dockerized Nginx reverse proxy setups.
* Using Nginx as a static file CDN.

---

## ✅ Summary

* Nginx is both **a web server and a reverse proxy** — efficient, flexible, and production-grade.
* Serve static content directly and offload dynamic requests to backends.
* Always test (`nginx -t`) before reloading.
* Secure with HTTPS, caching, and rate limiting.
* Lightweight, predictable, and nearly indestructible — it’s the web’s quiet workhorse.

---

📄 **File path suggestion:**

```
docs/
└─ cheatsheets/
   └─ tools/
      └─ nginx/
         └─ quick-start.md
```

---

## 💻 16. Nginx in Developer Workflows (JetBrains & VS Code)

Nginx isn’t just a server you “deploy somewhere.”  
It’s a **local testing tool**, **reverse proxy in development**, and part of modern CI/CD pipelines.

---

### 🧩 JetBrains IDEs (IntelliJ IDEA, PyCharm, etc.)

| Feature | What It Does |
|----------|--------------|
| **File templates** | Built-in syntax highlighting for `nginx.conf` and `.conf` files. |
| **Remote deployment** | Sync `/etc/nginx/` or Docker configs via “Deployment → Remote Host”. |
| **Before launch tasks** | Run `nginx -t` automatically to test config before restarting. |
| **Docker integration** | Configure Nginx container services directly in IDE Services tab. |

**Pro tip:**  

In JetBrains, you can make a *Run Configuration* that executes:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

This gives you one-click config validation + reload directly from the IDE.

---

### 🧠 VS Code Integration

VS Code can act as your Nginx control panel with the right extensions.

**Recommended setup:**

* **Nginx Configuration Language** → syntax + linting
* **Nginx Snippets** → quick templates
* **Docker Extension** → manage running containers
* **REST Client** → test API endpoints proxied through Nginx

You can run Nginx locally for frontend-backend routing:

```bash
docker run -d \
  --name dev-nginx \
  -p 8080:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:latest
```

---

## 🧰 17. Nginx + Docker Compose in Local Development

A clean, composable setup to proxy requests between your frontend and backend:

```yaml
version: "3.9"
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend
      - frontend

  frontend:
    build: ./frontend
    expose:
      - "3000"

  backend:
    build: ./backend
    expose:
      - "8080"
```

Then `nginx.conf`:

```nginx
events {}

http {
  server {
    listen 80;

    location / {
      proxy_pass http://frontend:3000;
    }

    location /api/ {
      proxy_pass http://backend:8080;
    }
  }
}
```

✅ **Benefits:**

* Unified local environment — no cross-origin chaos.
* Hot reload compatible (mount local code).
* Easier to mirror staging/production later.

---

## 🚀 18. Staging vs. Production Patterns

| Environment    | Goal                                          | Typical Nginx Role               |
| -------------- | --------------------------------------------- | -------------------------------- |
| **Local**      | Simulate routing and test caching/proxy rules | Run via Docker or native install |
| **Staging**    | Mimic production routing and SSL              | Reverse proxy + TLS              |
| **Production** | Serve static content + proxy dynamic requests | Load balancer + cache layer      |

Example split configs:

```
nginx/
 ├─ nginx.conf           # global settings
 ├─ conf.d/
 │   ├─ dev.conf         # local proxy setup
 │   ├─ staging.conf     # SSL, rate limiting
 │   └─ production.conf  # caching, load balancing
```

---

## 🧩 19. Common CI/CD Integrations

**Build pipeline (Dockerized):**

```yaml
# Dockerfile
FROM nginx:latest
COPY nginx.conf /etc/nginx/nginx.conf
COPY dist/ /usr/share/nginx/html
```

**GitHub Actions snippet:**

```yaml
- name: Build & Push Nginx Image
  run: |
    docker build -t ghcr.io/user/app-nginx:${{ github.sha }} .
    docker push ghcr.io/user/app-nginx:${{ github.sha }}
```

**Deployment example (Docker Swarm / Kubernetes):**

```yaml
# Swarm stack.yml
services:
  nginx:
    image: ghcr.io/user/app-nginx:latest
    ports:
      - "80:80"
      - "443:443"
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
```

---

## 🧠 20. Best Practices & Developer Habits

### Configuration Hygiene

* Keep production configs **readonly** and version-controlled.
* Split by domain or role — one file per app.
* Use includes:

  ```nginx
  include /etc/nginx/conf.d/*.conf;
  ```

### Developer sanity checklist

* Always run `nginx -t` before reload.
* Use `$host` and `$remote_addr` headers when proxying.
* Don’t run with root inside containers (use `nginx` user).
* Redirect all HTTP to HTTPS — even locally, if possible.
* Keep logs rotated (`logrotate` or Docker log limits).

### IDE habit

* Format configs automatically before commit.
* Use pre-commit hooks to validate syntax:

  ```bash
  nginx -t -q || exit 1
  ```

---

## ✅ Summary (Developer Edition)

* Integrate Nginx directly in IDE or Compose — no manual SSH needed.
* Test configs automatically before reloads.
* Run the same Nginx image locally and in production for consistency.
* Version-control your `.conf` files like code — because they *are* code.
* Treat Nginx as your **traffic controller**, not just a web server.
