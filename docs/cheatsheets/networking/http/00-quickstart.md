---
title: Quick Start
tags: 
    - http
    - api
    - rest
    - sessions
    - tokens
    - cookies
    - authentication
    - security
summary: A compact, no-fluff HTTP reference for daily use.
aliases:
  - HTTP Quick Start
---

# ⚡ HTTP — Quick Start

A speed-run of the web’s lingua franca. Keep this handy; split it later when you outgrow it.

---

## 🧭 Mental Model

HTTP is like sending letters through a global post office:

* You (the **client**) write a **request letter** with headers and maybe a body.
* The **server** reads it and replies with a **response letter**.
* Each exchange is independent — no long-term memory unless you bring cookies.

That’s why you often combine it with:

* **Sessions / Tokens** → to maintain identity
* **TLS (HTTPS)** → to keep the mail private
* **Caching** → to avoid sending the same letter again

---

## 1) What HTTP Is (and Isn’t)

* **HTTP** = **H**yper**T**ext **T**ransfer **P**rotocol — rules for how a **client** talks to a **server**.
* **Stateless**: each request stands alone. Servers don’t “remember” you unless you give them state (cookies, tokens).
* **Text-first**: start line → headers → optional body. Easy to read, easy to debug.
* **HTTPS = HTTP over TLS** for privacy + integrity.

```
Client ──request──> Server
       <─response──
```

---

## 2) The Shape of a Request

```
GET /path?key=value HTTP/1.1
Host: example.com
User-Agent: curl/8.0
Accept: application/json
Authorization: Bearer <token>

<body is optional>
```

Key parts:

* **Method** (GET/POST/PUT/DELETE/…)
* **Target** (`/path?query`)
* **Version** (HTTP/1.1 or HTTP/2/HTTP/3)
* **Headers** (metadata)
* **Body** (optional; usually with POST/PUT/PATCH)

### Minimal cURL

```bash
# Simple GET
curl https://example.com/api/items

# GET with query + headers
curl -H "Accept: application/json" \
     "https://example.com/search?q=books&page=2"

# POST JSON
curl -X POST -H "Content-Type: application/json" \
     -d '{"title":"Dune"}' https://example.com/api/books
```

---

## 3) The Shape of a Response

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 27
Cache-Control: max-age=3600

{"message":"hello, world"}
```

* **Status line**: version + status code + reason phrase
* **Headers**: metadata about the payload/caching/etc.
* **Body**: bytes (JSON, HTML, image, zip…).

---

## 4) Methods (What You’re Asking For)

| Method  | Idempotent? | Has body? | Typical use                                    |
| ------- | ----------- | --------- | ---------------------------------------------- |
| GET     | ✅           | ❌         | Fetch a resource (no side-effects).            |
| HEAD    | ✅           | ❌         | GET without body (check headers/size).         |
| POST    | ❌           | ✅         | Create / command / non-idempotent actions.     |
| PUT     | ✅           | ✅         | Replace a resource entirely at the target URI. |
| PATCH   | ❌(usually)  | ✅         | Partial update of a resource.                  |
| DELETE  | ✅           | (opt)     | Remove a resource.                             |
| OPTIONS | ✅           | (opt)     | What can I do here? Used in CORS preflights.   |

> **Idempotent** = same request repeated → same result on the server.

---

## 5) Status Codes (Quick Map)

### 2xx — Success

* **200 OK** — here’s your thing.
* **201 Created** — new resource made; often returns `Location`.
* **204 No Content** — success, no body (e.g., DELETE).

### 3xx — Redirection

* **301 Moved Permanently**
* **302 Found** (temporary)
* **303 See Other** — after POST, go GET this URL.
* **307/308** — redirect but **don’t** change method/body.

### 4xx — Client Errors

* **400 Bad Request** — malformed request.
* **401 Unauthorized** — missing/invalid auth.
* **403 Forbidden** — authenticated but not allowed.
* **404 Not Found**
* **409 Conflict** — version mismatch, duplicate data.
* **422 Unprocessable Content** — validation failed.
* **429 Too Many Requests** — rate-limited.

### 5xx — Server Errors

* **500 Internal Server Error**
* **502 Bad Gateway**
* **503 Service Unavailable**
* **504 Gateway Timeout**

---

## 6) Headers That Actually Matter (Daily Use)

### Request headers

* `Host:` domain name (mandatory in HTTP/1.1).
* `Accept:` formats you’ll accept.
* `Authorization:` bearer token or basic credentials.
* `Content-Type:` body’s media type.
* `If-None-Match:` ask only if changed (ETag).
* `If-Modified-Since:` ask only if newer than date.

### Response headers

* `Content-Type:` what’s in the body.
* `Cache-Control:` caching policy.
* `ETag:` body fingerprint for change detection.
* `Last-Modified:` timestamp of resource.
* `Vary:` which headers affect caching.
* `Set-Cookie:` session or tracking data.
* `Location:` for redirects or newly created resources.

---

## 7) Caching (Speed Without Lies)

**Goal:** serve from cache when safe, skip server work, save bandwidth.

* **Freshness**

  * `Cache-Control: max-age=600` → cache 10 minutes.
  * `no-store` = never save; `no-cache` = revalidate first.
* **Revalidation**

  * `If-None-Match` + `ETag` or `If-Modified-Since` + `Last-Modified`.
  * Unchanged → `304 Not Modified`.
* **User data**

  * `Cache-Control: private, no-store`

---

## 8) Content Negotiation

Client says what it wants; server chooses best match.

Example:

```
Accept: application/json, text/html;q=0.8
```

Response:

```
Content-Type: application/json
```

---

## 9) CORS (Cross-Origin Resource Sharing)

* Browsers block JS calls across domains by default.
* Server enables access with headers:

```
Access-Control-Allow-Origin: https://yourapp.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

Complex requests trigger an **OPTIONS preflight**.

---

## 10) HTTP Versions at a Glance

* **HTTP/1.1:** text-based, single request per connection.
* **HTTP/2:** binary, multiplexed, header compression (HPACK).
* **HTTP/3:** runs on QUIC (UDP), faster handshakes, fewer delays.

Negotiated automatically; you don’t code to versions.

---

## 🚀 Performance & Modern HTTP Practices

* **Compression:** `Content-Encoding: gzip` or `br`.
* **Connection reuse:** `Connection: keep-alive`.
* **Streaming:** chunked transfer for large payloads.
* **CDNs:** act as distributed HTTP caches.
* **Caching layers:** browser → proxy → CDN → origin.

---

## 🔐 Security at a Glance

* Always **use HTTPS** — plain HTTP is obsolete.
* **Validate input** even from “trusted” clients.
* **Don’t leak stack traces or system info** in errors.
* Use secure cookie flags:

  * `SameSite=Strict; Secure; HttpOnly`
* Defensive headers:

  ```
  X-Content-Type-Options: nosniff
  X-Frame-Options: DENY
  Referrer-Policy: no-referrer
  Content-Security-Policy: default-src 'self'
  ```

---

## 11) Debug Like You Mean It

```bash
# Show response headers only
curl -I https://example.com

# Verbose
curl -v https://example.com

# Follow redirects
curl -L https://example.com

# Conditional GET
curl -H 'If-None-Match: "abc123"' -I https://example.com/resource
```

---

## 12) Tiny REST API Primer

* **Resources** get **URIs**: `/users/42`, `/orders/77/items`.
* **Methods** model intent: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`.
* **Status codes** = communication contract.
* **Representation** is usually JSON:

  * `Content-Type: application/json`
* **Error body:** structured JSON with `code`, `message`, `fields`.

---

## 13) Minimal “Must-Know” List

* **Methods:** GET / POST / PUT / PATCH / DELETE
* **Codes:** 200 / 201 / 204 / 301 / 304 / 400 / 401 / 403 / 404 / 409 / 422 / 429 / 500
* **Headers:** Content-Type / Accept / Authorization / Cache-Control / ETag / Vary / Location
* **Caching:** max-age / ETag + If-None-Match / 304

---

## 14) Java & Spring Quick Hooks

```java
@GetMapping("/books/{id}")
@PostMapping(value="/books", consumes="application/json", produces="application/json")
@PutMapping("/books/{id}")
@DeleteMapping("/books/{id}")

return ResponseEntity.ok()
    .header("Cache-Control", "public, max-age=60")
    .eTag(hash)
    .body(dto);
```

---

### 🧩 Pocket Glossary

* **Idempotent:** redoing the same call yields the same server state.
* **Safe:** doesn’t change server state (GET, HEAD).
* **Origin:** scheme + host + port combo.
* **Payload:** the body of a request or response.

---

### 🧱 When You Split This File Later

Break into:

```
http-basics.md
methods.md
headers.md
status-codes.md
caching.md
security.md
```

Keep this file as your top-level “map” of HTTP knowledge.

