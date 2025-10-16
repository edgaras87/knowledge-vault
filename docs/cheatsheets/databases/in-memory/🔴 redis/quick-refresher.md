---
title: quick-refresher
tags: [redis, in-memory, database, cache, docker, spring-boot, pubsub, streams]
summary: core concepts, commands, data structures, Docker setup, app integration, persistence modes, patterns, and best practices.
---

# 🔴 Redis: From Basics to Real-World Usage

Redis (Remote Dictionary Server) is a **lightning-fast in-memory data store** used as a cache, message broker, and lightweight database.  
It stores data as **key-value pairs**, keeps it in RAM for near-instant access, and can optionally persist it to disk.

It’s often used alongside PostgreSQL: Postgres for durability, Redis for speed.

---

## ⚙️ 1. What Redis Actually Does

Redis keeps everything in **memory** but optionally syncs to disk (snapshots or append logs).  
It supports complex data types, pub/sub messaging, and atomic operations — all with sub-millisecond latency.

| Use Case | Example |
|-----------|----------|
| **Cache** | Store frequently accessed DB results. |
| **Session Store** | Track logged-in users. |
| **Queue / Stream** | Background jobs, real-time feeds. |
| **Rate Limiter** | Count requests per user/IP. |
| **Pub/Sub** | Event notification between services. |

👉 In short: Redis trades *persistence* for *speed*, but can be configured for both.

---

## 🧱 2. Core Concepts

| Concept | Description |
|----------|-------------|
| **Key** | Identifier for a piece of data (string). |
| **Value** | Data stored under a key — can be string, hash, list, set, etc. |
| **TTL** | Time-to-live (expiration). |
| **Persistence** | Data saved to disk (RDB snapshots or AOF logs). |
| **Pub/Sub** | Publisher sends messages to channels; subscribers receive them. |
| **Database index** | Redis has logical DBs numbered 0–15 by default. |

---

## ⚡ 3. Basic Commands (Quick Reference)

```bash
redis-cli                   # Start CLI
ping                        # → PONG
set key "value"             # store
get key                     # retrieve
del key                     # delete
exists key                  # check if exists
expire key 60               # expire after 60s
keys *                      # list all keys (use with caution)
flushall                    # delete everything
````

---

## 🧩 4. Data Structures and Examples

### Strings

```bash
set greeting "Hello"
get greeting
incr counter
decr counter
```

### Hashes (key → fields)

```bash
hset user:1 name "Alice" email "alice@example.com"
hgetall user:1
hget user:1 email
```

### Lists (ordered queue)

```bash
lpush queue job1
rpush queue job2
lrange queue 0 -1
lpop queue
```

### Sets (unique values)

```bash
sadd users "edgaras" "alice"
smembers users
sismember users "bob"
```

### Sorted Sets (ranking)

```bash
zadd leaderboard 100 "alice" 95 "bob"
zrevrange leaderboard 0 -1 WITHSCORES
```

### Pub/Sub

```bash
SUBSCRIBE news
PUBLISH news "Server update complete!"
```

---

## 🧠 5. Persistence Modes

Redis can persist data in two main ways:

| Mode                       | Description                                                        |
| -------------------------- | ------------------------------------------------------------------ |
| **RDB (Snapshot)**         | Saves full dataset at intervals (default). Fast, minimal overhead. |
| **AOF (Append Only File)** | Logs every operation — safer but slower.                           |
| **Hybrid**                 | Combines both for speed + durability.                              |

Example in `redis.conf`:

```
save 900 1
save 300 10
save 60 10000
appendonly yes
```

---

## 🧰 6. Running Redis with Docker

```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:7
```

Check connection:

```bash
docker exec -it redis redis-cli ping
```

With password:

```bash
docker run -d \
  -p 6379:6379 \
  -e REDIS_PASSWORD=secret \
  redis:7 \
  redis-server --requirepass secret
```

---

## 🧩 7. Docker Compose Example

```yaml
version: "3.9"
services:
  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5

  backend:
    build: ./backend
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379

volumes:
  redis_data:
```

---

## 💾 8. Redis + Application Integration (Spring Boot Example)

`application.properties`:

```properties
spring.data.redis.host=redis
spring.data.redis.port=6379
spring.data.redis.password=secret
```

Simple cache usage with Spring:

```java
@Cacheable("users")
public User getUserById(Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

Make sure caching is enabled:

```java
@EnableCaching
@SpringBootApplication
public class App {}
```

---

## 📡 9. Monitoring and Admin Commands

```bash
info                   # show metrics
dbsize                 # number of keys
monitor                # live command log
config get *           # view config
config rewrite          # persist config changes
slowlog get            # show slow commands
```

For real-time dashboards:

* [RedisInsight](https://redis.io/docs/ui/insight/)
* `redis-exporter` for Prometheus / Grafana metrics.

---

## 🔒 10. Security & Performance Best Practices

* **Set a password** (`requirepass secret`) — Redis is open by default.
* **Bind to localhost** or internal networks (`bind 127.0.0.1`).
* Disable `FLUSHALL` and `CONFIG` commands in production.
* Use **connection pooling** for app clients.
* For persistence: prefer **AOF + fsync every second**.
* Enable `maxmemory` and `maxmemory-policy allkeys-lru` for safe eviction.

Example snippet:

```
maxmemory 512mb
maxmemory-policy allkeys-lru
```

---

## 💻 11. IDE & Tool Integration

### JetBrains IDEs

* Use built-in **Database Tool Window** → Add Data Source → Redis.
* Visualize keys, TTLs, and values directly.
* Supports `EVAL` and Lua scripting.

### VS Code

Recommended extensions:

* **Redis Explorer** → browse keys, TTLs, and memory usage.
* **REST Client** → test APIs that interact with Redis.
* **.env files** → store connection secrets.

Example `.env`:

```
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=secret
```

---

## 🚀 12. CI/CD Integration Example

GitHub Actions — Redis as test dependency:

```yaml
services:
  redis:
    image: redis:7
    ports: ['6379:6379']
    options: >-
      --health-cmd="redis-cli ping"
      --health-interval=5s --health-retries=5
steps:
  - uses: actions/checkout@v4
  - name: Run integration tests
    env:
      REDIS_HOST: localhost
      REDIS_PORT: 6379
    run: ./gradlew test
```

---

## 🧩 13. Real-World Patterns

| Pattern           | Description                                                  | Example                       |
| ----------------- | ------------------------------------------------------------ | ----------------------------- |
| **Cache-Aside**   | App reads from Redis; on miss, fetches DB + stores in Redis. | Common with Spring or Django. |
| **Write-Through** | Writes go to Redis and DB simultaneously.                    | Ensures consistency.          |
| **Pub/Sub**       | Services communicate via Redis channels.                     | Real-time notifications.      |
| **Streams**       | Event queue with consumer groups.                            | Great for jobs, analytics.    |

Example stream usage:

```bash
XADD jobs * type "email" user "alice"
XREADGROUP GROUP workers 1 COUNT 1 STREAMS jobs >
```

---

## 🧠 14. Troubleshooting

| Issue               | Fix                                       |
| ------------------- | ----------------------------------------- |
| Redis not reachable | Check port 6379 and container health.     |
| Keys disappear      | TTL expired or memory eviction triggered. |
| “NOAUTH” error      | Set password in config and client.        |
| High latency        | Tune `maxmemory` + eviction policy.       |
| Data not persistent | Enable `appendonly yes`.                  |

---

## ✅ 15. Summary

* Redis = speed and simplicity — an in-memory data store with persistence options.
* Ideal for caching, pub/sub, queues, and rate limiting.
* Use Docker for easy setup and Compose for multi-service integration.
* Always secure, monitor, and limit memory.
* Combine with PostgreSQL for the best of both worlds: durability + velocity.

---

📄 **File path suggestion:**

```
docs/
└─ cheatsheets/
   └─ tools/
      └─ redis/
         └─ quick-refresher.md
```

