---
title: Cookies
tags:
  - http
  - api
  - rest
  - sessions
  - tokens
  - cookies
  - authentication
  - security
summary: A quick reference on how HTTP cookies work, their attributes, security best practices, and practical examples.
aliases:
  - Cookies
---

# 🍪 HTTP Cookies — Quick Refresher

Tiny text packets that let the web remember.

---

## 1) What a Cookie Is

A **cookie** is a small key–value pair that the **server** tells the **browser** to store.
The browser automatically sends it back with every request to the same origin.

Cookies exist because **HTTP is stateless** — servers forget who you are between requests.
Cookies give the illusion of continuity.

```
Server → Set-Cookie: sessionId=abc123; Secure; HttpOnly
Browser → Cookie: sessionId=abc123
```

They’re the oldest and most universal state mechanism in HTTP.

---

## 2) Anatomy of a Cookie

Each cookie line looks like this:

```
Set-Cookie: key=value; Attribute1; Attribute2=...
```

### Example

```
Set-Cookie: theme=dark; Path=/; Max-Age=3600; SameSite=Lax
```

The browser then includes it:

```
Cookie: theme=dark
```

---

## 3) Where Cookies Live

* **In browsers:** limited (~20 per domain, 4 KB each).
* **By origin:** tied to domain + path + scheme (HTTP/HTTPS).
* **Automatically sent:** for same-origin requests, no client code needed.

---

## 4) Common Attributes and Their Meaning

| Attribute    | Purpose                      | Typical Value / Example                 |
| ------------ | ---------------------------- | --------------------------------------- |
| **Domain**   | Which host(s) can receive it | `Domain=example.com`                    |
| **Path**     | Restrict to subpath          | `Path=/api`                             |
| **Expires**  | Absolute expiry time         | `Expires=Wed, 16 Oct 2025 07:00:00 GMT` |
| **Max-Age**  | Relative expiry in seconds   | `Max-Age=3600`                          |
| **Secure**   | Send only via HTTPS          | `Secure`                                |
| **HttpOnly** | Hide from JavaScript         | `HttpOnly`                              |
| **SameSite** | Limit cross-site sending     | `Strict`, `Lax`, `None`                 |

---

## 5) Lifecycle

1. **Server sets** the cookie via `Set-Cookie` header.
2. **Browser stores** it until it expires or user clears it.
3. **Browser sends** it back in future requests matching domain/path/scheme.
4. **Server reads** it from `Cookie:` header.

To delete:

```
Set-Cookie: sessionId=deleted; Max-Age=0
```

---

## 6) Session vs Persistent Cookies

| Type                  | Description                              | Lifetime                    |
| --------------------- | ---------------------------------------- | --------------------------- |
| **Session cookie**    | No `Expires`/`Max-Age`; stored in memory | Removed when browser closes |
| **Persistent cookie** | Has expiration                           | Survives browser restarts   |

Session cookies → logins
Persistent cookies → “remember me”

---

## 7) Security Essentials

### ✅ Always use:

```
Set-Cookie: sessionId=abc123; Secure; HttpOnly; SameSite=Strict
```

### Key points

* `Secure`: no plaintext transmission.
* `HttpOnly`: prevents JS theft (XSS).
* `SameSite`: defends against CSRF.
* Don’t store sensitive data directly inside cookies.
* Avoid overly broad domains (`Domain=.example.com` shares across subdomains).

---

## 8) SameSite Explained Clearly

| Mode       | Behavior                         | Use Case                                                      |
| ---------- | -------------------------------- | ------------------------------------------------------------- |
| **Strict** | Sent only from same site         | Most secure (auth cookies)                                    |
| **Lax**    | Sent on top-level navigations    | Default for browsers                                          |
| **None**   | Sent everywhere, even cross-site | Must combine with `Secure`; used for third-party integrations |

Example:

```
Set-Cookie: auth=abc123; SameSite=None; Secure
```

---

## 9) Cookies in APIs and SPAs

Browsers attach cookies automatically only for same-origin requests.
When making cross-origin API calls, you must explicitly enable credential sharing.

### Client (JavaScript)

```js
fetch("https://api.example.com/data", {
  credentials: "include"
});
```

### Server (API)

```
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: https://app.example.com
```

This handshake enables cookie-based auth across origins.

---

## 9.5) Modern Alternatives

While cookies remain the **native way browsers persist state**, modern systems often use other approaches — especially for APIs and mobile apps.

### Session Tokens

Instead of storing a session ID in a cookie, the server issues a **token** (like a signed string) that the client stores manually (e.g., in `localStorage`) and sends in headers such as:

```
Authorization: Bearer <token>
```

### JWT (JSON Web Token)

A **self-contained, signed token** that carries claims — user ID, roles, expiration.
Used for stateless authentication — the server doesn’t need to remember anything.
But with power comes danger: once issued, it’s valid until expiry, so **revocation is harder**.

### Server-Side Sessions

The classic approach — store session data on the **server**, and just send a session ID via cookie.
Simpler to invalidate, but less scalable in distributed systems.

**In short:**

* **Cookies** are the browser-native state glue.
* **Tokens** are the API-native evolution.
  You’ll often see **both working together** — cookies wrapping tokens for web clients, tokens alone for APIs and mobile apps.

---

## 10) Cookie Storage Alternatives

* **LocalStorage / SessionStorage** — manually managed by JS, not auto-sent.
* **Cookies** — automatic, integrated with HTTP.
* Use **HttpOnly cookies** for secrets; use storage APIs for preferences.

---

## 11) Practical Example — Login Flow

```http
POST /login
→ Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict

GET /dashboard
→ Cookie: sessionId=abc123
```

Browser sends the cookie automatically; server validates the session.

Logout:

```
Set-Cookie: sessionId=; Max-Age=0
```

---

## 12) Debugging & Inspection

### Using cURL

```bash
# Show cookies in response
curl -I -c cookies.txt https://example.com

# Send stored cookies
curl -b cookies.txt https://example.com/profile
```

### In Browser

Open **DevTools → Application → Cookies** to view, edit, or delete.

---

## 13) Java & Spring Example

### Setting

```java
ResponseCookie cookie = ResponseCookie.from("sessionId", "abc123")
    .httpOnly(true)
    .secure(true)
    .sameSite("Strict")
    .path("/")
    .maxAge(Duration.ofHours(1))
    .build();

return ResponseEntity.ok()
    .header(HttpHeaders.SET_COOKIE, cookie.toString())
    .body("ok");
```

### Reading

```java
@CookieValue("sessionId") String sessionId
```

---

## 14) Quick Reference Table

| Purpose              | Header       | Example                                 |
| -------------------- | ------------ | --------------------------------------- |
| Set new cookie       | `Set-Cookie` | `Set-Cookie: user=edgaras; Max-Age=600` |
| Send existing cookie | `Cookie`     | `Cookie: user=edgaras`                  |
| Delete cookie        | `Set-Cookie` | `Set-Cookie: user=; Max-Age=0`          |

---

## 15) Mental Checklist

* Always add `Secure; HttpOnly; SameSite`.
* Keep cookies small.
* Avoid sensitive info inside.
* Delete or expire aggressively.
* Treat them as *credentials*, not just data.

---

## 16) In a Sentence

Cookies are the quiet workhorses of the web — small, loyal, and dumb.
They make a stateless protocol feel personal, but trust them only when you’ve set the rules.

