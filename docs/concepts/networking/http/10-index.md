---
title: Starter
tags: 
    - http
    - api
    - rest
    - sessions
    - tokens
    - cookies
    - authentication
    - security
summary: A guided path through HTTP, cookies, and tokens — from stateless requests to modern authentication. 
---


# 🌍 Web & HTTP — Starter Guide

### 🧩 Purpose

This folder collects everything you need to truly understand **how the web communicates and remembers** — from raw HTTP to modern authentication.

Each file stands alone, but reading them **in order** reveals the full story of how the web evolved from *stateless requests* to *secure, identity-aware systems*.

---

## ️⃣ Read Order

| Step   | File                                      | Theme                        | Why it comes here                                                                                                            |
| ------ | ----------------------------------------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **1.** | [[20-http-foundations|HTTP Basics]]          | **HTTP fundamentals**        | The physics of the web: requests, responses, headers, and why HTTP is stateless by design.                                   |
| **2.** | [[30-state-evolution| HTTP State Evolution]] | **How state was invented**   | Explains *why* cookies and tokens exist at all — the journey from “forgetful HTTP” to “authenticated APIs.”                  |
| **3.** | [[40-cookies| Cookies]]                    | **Browser-native state**     | Deep dive into cookies: how servers create them, how browsers send them, and how they keep classic sessions alive.           |
| **4.** | [[50-tokens| Tokens]]                     | **Stateless authentication** | Modern replacement for sessions — how APIs and mobile apps use tokens (JWT, OAuth2) to prove identity without server memory. |

---

## 🔄 Conceptual Flow

```
HTTP (stateless)
      ↓
Need for continuity (state)
      ↓
Cookies (server remembers)
      ↓
Tokens (client proves)
```

This mirrors the real evolution of the web:
from simple document delivery → to remembering users → to trustless, scalable identity systems.

---

## 🧠 Reading Tips

* **Move slow.** Each file adds a layer of understanding — don’t rush.
* **Visualize the requests.** Think like both the browser and the server; it makes “state” click instantly.
* **Focus on concepts before frameworks.** Spring, Node, or Python are just different dialects of the same logic.
* **Experiment with `curl`.** Seeing headers in action (like `Set-Cookie` or `Authorization`) cements the idea better than any paragraph.

---

## 🗺️ After Finishing

You’ll be able to:

* Explain what “stateless” actually means
* Understand cookies vs sessions vs tokens
* Recognize why tokens dominate modern systems
* Map all of this to frameworks like **Spring Boot**, **Express**, or **FastAPI**

Next natural steps:

* [**oauth-oidc.md**]() → how delegated auth works (Google, GitHub login)
* [**spring-security-jwt.md**]() → how Spring implements modern token auth
* [**auth-ops-checklist.md**]() → production hygiene: rotation, revocation, key management

---

### 🧩 In One Sentence

Start with **HTTP — how messages work**,
end with **tokens — how identity travels**.
Everything between those two explains how the web learned to *remember you safely*.

