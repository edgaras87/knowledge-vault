---
title: State (Evolution)
tags:
  - http
  - api
  - rest
  - sessions
  - tokens
  - cookies
  - authentication
  - security
  - spring
summary: A conceptual overview of how web state evolved from cookies to tokens, and how Spring reflects that journey.
aliases:
  - HTTP State Evolution
---

# 🌐 Web State Evolution (told through Spring’s lens)

How the web learned to *remember* — and how that memory evolved from cookies to tokens, from servers to clients.


**See also:** [[state|HTTP State Reference]]


---

## 1) Why Web State Exists

HTTP was designed to be **stateless** — every request is a blank slate.
Without extra mechanisms, the server can’t tell whether two requests come from the same user.

To make web experiences persistent — logins, carts, dashboards — we need a **state bridge** between requests.
That bridge has changed over time, but the problem never did.

---

## 2) Cookies — The Original Memory Hack

A **cookie** is a tiny key–value pair that a server asks the browser to keep and send back with future requests.

```
Server → Set-Cookie: sessionId=abc123
Browser → Cookie: sessionId=abc123
```

Cookies make HTTP *feel* stateful, but they’re just **labels**, not memory.
The actual data lives on the **server**, keyed by that label:

```
sessionId=abc123 → user=Edgaras, cart=3 items
```

Think of it as a coat-check system:

* You (the browser) keep the ticket.
* The server keeps your coat (the session data).

When you return, you hand in the ticket; the server finds your coat.

If either side forgets (cookie cleared or session expired), you start fresh.

---

## 3) Why the Client Can’t Be Trusted

Cookies are stored client-side, so they can be read, changed, or deleted.
That’s why cookies must be protected with strict attributes:

| Attribute  | Protects Against  | Description                 |
| ---------- | ----------------- | --------------------------- |
| `HttpOnly` | JavaScript access | JS can’t read or modify it  |
| `Secure`   | Network sniffing  | Only sent via HTTPS         |
| `SameSite` | CSRF attacks      | Controls cross-site sending |

Servers must **never trust cookie data blindly** — it’s a reference, not proof.

---

## 4) The Stateful Model — Server Remembers You

### Example: Spring MVC

Old-school web applications used **server-side sessions**.

```
POST /login → Set-Cookie: JSESSIONID=xyz123
```

Spring stores that session internally:

```
xyz123 → userId=42, username="edgaras"
```

Next request:

```
GET /profile
Cookie: JSESSIONID=xyz123
```

Spring restores your identity automatically via `HttpSession`.

Code sketch:

```java
session.setAttribute("user", user);
User user = (User) session.getAttribute("user");
```

This model is **stateful** — the server remembers each user’s state in memory or a shared store (like Redis).

---

## 5) The Stateless Model — Client Proves Itself

Modern REST APIs abandoned server memory.
Instead of “the server remembers you,” we now do **“you prove who you are each time.”**

### Example: Spring Boot (REST API)

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
```

The token (e.g., JWT) is:

* Signed by the server
* Self-contained (includes user info, expiration, roles)
* Verifiable without server memory

No lookup tables. No `HttpSession`.
Every request is independent and verifiable.

This makes scaling simple — any server can validate the token.

---

## 6) Why This Change Happened

The shift wasn’t stylistic — it was infrastructural.

Old model (sessions):

* Works for single servers.
* Breaks in load-balanced or cloud environments (session stickiness issues).

New model (tokens):

* Works with multiple instances.
* Stateless and cache-friendly.
* Portable across APIs, apps, and devices.

Tokens are ideal for **distributed**, **multi-client** systems.
Cookies still shine in **browser-native**, **single-origin** web apps.

---

## 7) Hybrid World — Cookies Meet Tokens

Many modern systems combine both ideas:

* **Browser clients:** store JWT in an **HttpOnly cookie** for security
* **Mobile / API clients:** store JWT manually and send via `Authorization` header

Same token, different transport.

Cookies are now often just the *delivery vehicle* for tokens.

---

## 8) Cross-Language Equivalents

### 🧩 Java / Spring

* Old: `HttpSession` + `JSESSIONID`
* New: JWT / OAuth2 in `Authorization` header
* Advanced: WebFlux (reactive, stateless)

### ⚙️ Node.js / Express

* Old: `express-session` + `connect-redis`
* New: `jsonwebtoken` + stateless JWT validation
* Middleware-driven auth; typically one-liners

### 🐍 Python

* **Django (classic)**: sessionid cookie → DB-backed session
* **Flask**: `flask-session` (stateful) or `flask-jwt-extended` (stateless)
* **FastAPI**: modern, JWT-first design

### ⚡ JavaScript Frontends

* **React / Vue / Angular** store tokens in `localStorage` or cookies
* Send them as `Authorization: Bearer ...`
* Tokens make frontends independent from backend state

Each ecosystem faces the same choice:

> *Do we store memory on the server (stateful) or carry proof on the client (stateless)?*

---

## 9) Concept Table — The Evolution in One View

| Era         | Approach                    | Who Remembers You | Stored Where          | Still Used? | Example                 |
| ----------- | --------------------------- | ----------------- | --------------------- | ----------- | ----------------------- |
| 1990s–2010s | **Sessions + Cookies**      | Server            | Memory / DB           | ✅ Yes       | Django, Spring MVC      |
| 2010s–2020s | **Tokens (JWT/OAuth2)**     | Client            | Header / localStorage | ✅ Yes       | Spring Boot, FastAPI    |
| 2020s→      | **Hybrid Cookies + Tokens** | Both              | Cookie-wrapped JWT    | ✅ Growing   | SPAs, Cloud-native apps |

---

## 10) Core Insight

Cookies gave the web its first memory,
but tokens gave it *freedom* — freedom to scale, to move between servers, to talk to any device.

Spring just reflects that same journey:

* From **JSESSIONID** → to **JWT** → to **OAuth2/OpenID Connect**.

The evolution wasn’t just technical — it was philosophical:
moving from *the server knows you* to *you can prove who you are.*

---

## 11) Practical Developer Takeaways

* Understand **cookies and sessions** conceptually — they’re still vital for browsers.
* Build your APIs **statelessly** — each request should carry its own proof (token).
* Use **Spring Security + JWT/OAuth2** as your modern baseline.
* Learn **cross-language equivalents** — they all rhyme with Spring’s concepts.
* For scale: **state belongs at the edge (client)**, not in memory on the server.

---

## 12) In a Sentence

Cookies made the web personal.
Tokens made it planetary.

