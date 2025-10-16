---
title: State
tags:
  - http
  - api
  - rest
  - sessions
  - tokens
  - cookies
  - authentication
  - security
summary: A quick reference on how HTTP handles state with sessions, tokens, and cookies.
aliases:
  - HTTP State Reference
---



# 🍪 Sessions, Tokens & Cookies — Quick Refresher

How HTTP, a stateless protocol, learns to remember you.

**See also:** [[30-state-evolution|HTTP State Evolution]]

---

## 1) The Core Problem

HTTP by design has **no memory**.
Each request is independent — the server doesn’t know it’s *you* returning unless you tell it every time.

So to build logins, carts, dashboards, and “remember me” experiences, we need a **state mechanism** — something that links separate requests together.

Three main patterns exist:

| Mechanism    | Where State Lives             | Common Use                    |
| ------------ | ----------------------------- | ----------------------------- |
| **Sessions** | On the **server**             | Web apps, classic login       |
| **Cookies**  | On the **client (browser)**   | Store session IDs, small data |
| **Tokens**   | On the **client (API world)** | JWTs, OAuth access tokens     |

---

## 2) Sessions — The Old but Gold Approach

### 🧩 Idea

Store a “session record” on the **server** — usually a hash map like:

```
sessionId -> { userId: 42, role: "admin", cart: [...] }
```

The client just holds a random session ID:

```
Cookie: sessionId=abc123
```

The server looks it up on every request.

### ⚙️ How It Works

1. User logs in with username/password.
2. Server creates a session in memory or database.
3. Server sends the session ID to the client as a cookie.
4. Client sends it with every request.

### 💡 Pros

* Easy to invalidate (`delete sessionId` server-side).
* Works with simple cookies (no JWT complexity).
* Secure if server stores minimal data.

### ⚠️ Cons

* Doesn’t scale easily across multiple servers unless you share session storage (Redis, database, etc.).
* Not ideal for stateless APIs.

### Example

```http
POST /login
→ Set-Cookie: sessionId=abc123; HttpOnly; Secure

GET /profile
→ Cookie: sessionId=abc123
```

---

## 3) Cookies — The Tiny Carriers of State

Cookies are **small text blobs** (≈4KB each) stored by browsers per origin.

They automatically attach to every HTTP request to that site.

### 🍬 Example

```
Set-Cookie: theme=dark; Path=/; Max-Age=3600; SameSite=Lax
```

Browser then adds:

```
Cookie: theme=dark
```

### 🧠 Key Attributes

| Attribute             | Meaning                                     |
| --------------------- | ------------------------------------------- |
| **Domain**            | which host(s) can read it                   |
| **Path**              | restrict cookie to part of site             |
| **Expires / Max-Age** | when to delete it                           |
| **Secure**            | only send over HTTPS                        |
| **HttpOnly**          | JS can’t access it (prevents XSS stealing)  |
| **SameSite**          | controls cross-site sending (prevents CSRF) |

### 🔒 Security baseline

```http
Set-Cookie: sessionId=abc123; Secure; HttpOnly; SameSite=Strict
```

---

## 4) Tokens — Modern, Stateless Authentication

Tokens shift state **to the client**, so servers don’t store sessions.
They’re ideal for APIs, SPAs, and mobile apps.

The two main types:

| Token Type               | Format      | Where Stored          | Expiry                 |
| ------------------------ | ----------- | --------------------- | ---------------------- |
| **JWT (JSON Web Token)** | signed JSON | localStorage / cookie | short-lived (mins–hrs) |
| **Opaque Token**         | random ID   | server DB lookup      | flexible               |

### 🔧 JWT Example

Header.Payload.Signature — all Base64 encoded.

```json
{
  "sub": "42",
  "name": "Edgaras",
  "exp": 1737000000
}
```

JWTs are signed with a secret (HMAC) or private key (RSA).
The server verifies the signature — no DB lookup needed.

### Typical Flow

1. User logs in → server returns `access_token` + `refresh_token`.
2. Client stores them (preferably **in memory** or **httpOnly cookie**).
3. Sends token in every request:

   ```
   Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR...
   ```
4. When access token expires, client uses refresh token to get a new one.

### 💡 Pros

* Perfect for distributed / stateless backends.
* Scales easily.
* Easy to integrate with mobile / third-party clients.

### ⚠️ Cons

* Harder to invalidate before expiry (you can only blacklist).
* Token theft = full impersonation until expiry.

---

## 5) Cookies vs Tokens — The Real Difference

| Aspect       | Cookie Session                | Token-based                      |
| ------------ | ----------------------------- | -------------------------------- |
| State stored | Server-side                   | Client-side                      |
| Transport    | Automatic via browser         | Manual in `Authorization` header |
| Invalidation | Easy (delete on server)       | Harder (must track blacklists)   |
| Scalability  | Requires shared session store | Scales horizontally              |
| Works with   | Browser + HTML apps           | APIs, SPAs, mobile apps          |
| CSRF risk    | Yes                           | Minimal (if no cookies used)     |

---

## 6) Mixing Cookies & Tokens (Hybrid Pattern)

Modern secure apps often use **tokens inside cookies**.

```http
Set-Cookie: access_token=<JWT>; HttpOnly; Secure; SameSite=Strict
```

Benefits:

* Automatic sending like sessions.
* JWT’s stateless verification.
* HttpOnly flag protects from JS theft.

Used by major frameworks like **NextAuth**, **Spring Security**, and **Django Rest Framework**.

---

## 7) Refresh Tokens

To balance **security** and **convenience**, many systems use two tokens:

| Type              | Lifetime          | Purpose                      |
| ----------------- | ----------------- | ---------------------------- |
| **Access token**  | short (5–15 min)  | Used for API requests        |
| **Refresh token** | long (days–weeks) | Used to get new access token |

The refresh token is sent only to `/auth/refresh` endpoint — not every request.

---

## 8) CSRF & XSS — The Twin Threats

### Cross-Site Request Forgery (CSRF)

When attacker tricks browser into sending a cookie-authenticated request.
Prevent with:

* `SameSite=Strict` or `Lax` cookies.
* CSRF token in form submissions.
* Use of `Authorization` header instead of cookies for APIs.

### Cross-Site Scripting (XSS)

When injected JS steals tokens or cookies.
Prevent with:

* `HttpOnly` cookies.
* Strict Content-Security-Policy.
* Input sanitization.

---

## 9) Practical Example — Login Flow Comparison

### 🍪 Cookie-based (Sessions)

```
POST /login
→ Set-Cookie: sessionId=abc123; HttpOnly; Secure

GET /dashboard
→ Cookie: sessionId=abc123
```

### 🪙 Token-based (JWT)

```
POST /login
→ { "access_token": "eyJ...", "refresh_token": "eyJ..." }

GET /profile
→ Authorization: Bearer eyJ...
```

---

## 10) Quick Recap — What to Use When

| Use Case                       | Recommended                                   |
| ------------------------------ | --------------------------------------------- |
| Classic server-rendered site   | Sessions + Cookies                            |
| REST API / mobile app          | JWTs (access + refresh tokens)                |
| Microservices / distributed    | JWTs or opaque tokens with gateway validation |
| Sensitive data / high security | Short-lived tokens + HttpOnly cookies         |
| Public API                     | API keys (simple token variant)               |

---

## 11) Java & Spring Quick Hooks

### Session Example

```java
@PostMapping("/login")
public ResponseEntity<?> login(HttpSession session) {
    session.setAttribute("user", user);
    return ResponseEntity.ok().build();
}
```

### JWT Filter Example

```java
String authHeader = request.getHeader("Authorization");
if (authHeader != null && authHeader.startsWith("Bearer ")) {
    String token = authHeader.substring(7);
    Claims claims = jwtUtil.validateToken(token);
    // attach user info to security context
}
```

---

## 12) Pocket Glossary

* **Session:** server-side memory of a user’s state.
* **Cookie:** key-value pair stored on client, sent automatically with requests.
* **Token:** signed credential representing a user (e.g., JWT).
* **Refresh token:** used to issue new access tokens.
* **CSRF:** tricking browser into unintended requests.
* **XSS:** injecting scripts that run in a user’s browser.

---

## 13) Summary Thought

HTTP forgets.
Sessions, cookies, and tokens are three ways to make it *remember* — but each trades simplicity for scalability, and security for convenience.
Knowing when to **store state** and when to **trust statelessness** is what separates beginners from real backend engineers.

