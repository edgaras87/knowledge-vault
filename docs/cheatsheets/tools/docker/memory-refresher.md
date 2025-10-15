---
title: memory-refresher
tags: [docker, containers, devops, systemctl, compose, networking]
summary: managing Docker as a service, core concepts, commands, networking, multi-container apps with Compose, and troubleshooting.
---

# 🐳 Docker: From Service Basics to Multi-Container Setup

Docker packages software into **portable, isolated containers**.
Each container runs a process with its own filesystem, network, and dependencies.
Whether you’re controlling Docker as a Linux service or orchestrating multiple apps, its real power lies in **repeatability** and **consistency** across environments.

---

## ⚙️ 1. Controlling Docker as a Service (systemctl)

`systemctl` manages background services (daemons) on Linux via `systemd`.
Docker runs as one such service — you manage it like any other system process.

```bash
sudo systemctl start docker       # Start Docker service
sudo systemctl stop docker        # Stop Docker service
sudo systemctl restart docker     # Restart service
sudo systemctl status docker      # Check status and logs
sudo systemctl enable docker      # Start at boot
sudo systemctl disable docker     # Disable auto-start
systemctl list-units --type=service
````

👉 In short:
`systemctl` is the tool, `systemd` is the manager, and `docker` is the service you’re controlling.

---

## 🧱 2. Core Docker Concepts

| Concept                | Description                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------ |
| **Image**              | Blueprint for running software (e.g., `nginx:latest`, `mysql:8.4`). Immutable and versioned.     |
| **Container**          | A running instance of an image. Ephemeral — stops and vanishes unless data is stored externally. |
| **Volume**             | Persistent storage outside the container lifecycle (e.g., database data).                        |
| **Port Mapping**       | Bridge between host and container (`-p 8080:80` means host:8080 → container:80).                 |
| **Dockerfile**         | Script describing how to build an image. Defines base image, files, commands, and exposed ports. |
| **docker-compose.yml** | Declarative config for running multiple containers together (services, networks, volumes).       |

🧭 **Learn Docker in this order:**

1. Run a single container (`docker run nginx`)
2. Build your own image (write a Dockerfile)
3. Persist data with volumes
4. Connect containers with networks
5. Orchestrate everything using Compose

---

### 🔁 Container Lifecycle

```
Image → Container → Process → Exit → Removed
      ↑
   (built from Dockerfile)
```

A container is just a **process with boundaries** — lightweight isolation using Linux namespaces and cgroups.
It’s not a VM; it shares the host kernel but runs with its own view of the system.

---

## 🔧 Quick Command Reference

```bash
# Core commands
docker ps                   # running containers
docker ps -a                # all containers (including stopped)
docker images               # list local images
docker run -d -p 8080:80 nginx
docker exec -it <container> bash
docker logs -f <container>
docker stop <container>
docker rm <container>
docker system prune -a      # cleanup everything unused
```

---

## 📦 3. Container Networking (the hallway between containers)

Each container lives in its own isolated network namespace.
They can communicate when they share a **Docker network**.

**Bridge network (default):**
Containers can reach each other via IP, but not by name unless you define a user network.

**User-defined network (recommended):**

```bash
docker network create mynet
docker run -d --name db --network mynet mysql:8.4
docker run -d --name app --network mynet myapp
```

Now `app` can reach `db` simply at hostname `db:3306`.

**Ports and host access:**

```bash
docker run -p 8080:80 nginx
```

Maps port 8080 on your host to port 80 inside the container.

✅ **Best practice:**
Use Docker networks for container-to-container comms.
Expose ports only when the host or outside world must connect.

---

## 🧰 4. Example: Multi-Container App (Spring Boot + MySQL + Nginx)

```yaml
# docker-compose.yml
services:
  backend:
    build: ./backend
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/appdb?allowPublicKeyRetrieval=true&useSSL=false
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: app-pass
    ports:
      - "8080:8080"
    restart: unless-stopped

  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: app-pass
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -p$${MYSQL_ROOT_PASSWORD} || exit 1"]
      interval: 10s
      retries: 5

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    depends_on: [backend]

volumes:
  mysql_data:
```

Inside this network:

* `backend` talks to DB at **`mysql:3306`**
* `nginx` proxies to **`backend:8080`**
* You access via **`localhost:80`**

---

### 🏗️ Backend Dockerfile example

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### 🧩 Application properties (Spring Boot)

```properties
spring.datasource.url=jdbc:mysql://mysql:3306/appdb
spring.datasource.username=appuser
spring.datasource.password=app-pass
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

---

## 🧼 5. Housekeeping and Troubleshooting

### Cleanups

```bash
docker container prune        # remove stopped containers
docker image prune            # remove dangling images
docker system prune -a        # full cleanup (containers, images, volumes, networks)
```

### Debugging

```bash
docker ps -a                  # find exited containers
docker logs <name>            # read logs
docker inspect <name>         # metadata, IPs, mounts
docker exec -it <name> bash   # open shell inside container
```

---

## 🧠 Advanced Concepts (for later)

* **Image layers:** Every Dockerfile instruction adds a cached layer — build faster by ordering instructions wisely.
* **Isolation vs VMs:** Containers share the host kernel, making them lightweight and fast to start.
* **Security basics:** Avoid running containers as root in production; use minimal base images (e.g., `distroless`, `alpine`).
* **BuildKit:** Modern Docker build engine with parallel builds, inline caching, and secrets support (`DOCKER_BUILDKIT=1`).

---

## ✅ Summary

* Use **`systemctl`** to manage the Docker daemon.
* Use **`docker`** commands to run individual containers.
* Use **`docker-compose`** to run multi-container setups.
* Use **volumes** for persistence, **networks** for communication, and **healthchecks** for reliability.
* Start small, automate gradually, and remember: **containers are processes, not magic**.

---

### 🧭 Further expansion (when you’re ready)

Later, you can split this file into:

* `docker_basics.md`
* `docker_compose.md`
* `docker_networking.md`
* `docker_troubleshooting.md`




