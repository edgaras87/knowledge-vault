---
title: Tokens
tags:
  - http
  - api
  - rest
  - sessions
  - tokens
  - cookies
  - authentication
  - security
summary: A deep conceptual dive into what tokens really are, how statelessness works, and where state actually lives.
aliases:
  - Tokens
---

# 🧠 Understanding Tokens and Statelessness

---

### **What a Token Really Is**

A **token** is **proof of identity and permission**, not history.
It tells the server *who you are* and *what you’re allowed to do* — nothing more.

When you send a request with a token:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
```

the server verifies it and uses the claims inside to decide whether you’re authorized.

The server doesn’t “remember” you — it just **trusts the math** behind the token’s signature.

---

### **What “Stateless” Really Means**

In old, **stateful** models (cookies + sessions), the server remembered everything:

```
sessionId=abc123 → user=Edgaras, cart=3 items, lastPage=/checkout
```

Each request depended on the previous one.
That’s **stateful** — continuity stored in memory on the server.

In a **stateless** model (tokens):

* Each request is independent.
* The server doesn’t hold user memory between requests.
* If it needs context (cart, settings, history), it queries the **database**.

So “stateless” ≠ “no persistence.”
It means “no in-memory conversation” — every request brings everything needed to complete itself.

---

### **What’s Inside a Token**

Tokens carry only essential data (claims):

```json
{
  "sub": "42",          // who you are
  "role": "USER",       // what you can do
  "scope": "read:books",
  "exp": 1739678400     // when it expires
}
```

The server reads this info, verifies the signature, and makes decisions — without ever having to recall your previous state.

---

### **How the Server Sees It**

Each request is a self-contained event:

1. Receive the token.
2. Verify its signature (check it’s not forged).
3. Validate expiry and audience.
4. Use claims to authorize actions.
5. Fetch any needed data from persistent storage.

After responding, it forgets — no ongoing memory, no session list.

---

### **Permissions vs. History**

Tokens don’t store your personal history.
They store your **authority** — your access rights.

Think of it like an **access badge**:

* The badge shows your name, department, and which doors you can open.
* The guard doesn’t know what meetings you had yesterday.
* If your permissions change, you get a new badge.

That’s how modern auth works.

---

### **Where “State” Actually Lives**

All lasting data (orders, messages, progress, preferences) is stored in the **database**, linked to your user ID.
When you send a token, the server uses that ID to look up your data as needed.

So the *real* state lives in persistent storage — not in the session or the token.

---

### **Core Summary**

| Concept       | Meaning                                                   |
| ------------- | --------------------------------------------------------- |
| **Token**     | Digital proof of identity and permission                  |
| **Stateless** | Each request stands alone — no server memory between them |
| **State**     | Stored in the database, not in RAM                        |
| **Cookies**   | Used for stateful sessions (server remembers you)         |
| **Tokens**    | Used for stateless APIs (you prove yourself each time)    |

---

### **Analogy**

| Model              | Analogy                                                      | Who Keeps Memory |
| ------------------ | ------------------------------------------------------------ | ---------------- |
| Cookies / Sessions | You get a coat-check ticket; the server keeps your coat      | Server           |
| Tokens             | You carry a signed passport; the server just checks its seal | Client           |

---

### **In a Sentence**

> Cookies make the web remember you.
> Tokens make the web verify you.

---

```aiignore


```

---

# ❓ Key Q&A 

---

### 1️⃣ What the token *actually* does

A **token doesn’t carry your personal history**.
It carries **proof of identity and permission** — that’s it.

When you send it, the server checks:

> “Is this token valid, and what does it allow this user to do?”

It’s like flashing an ID badge to get into a building:

* The guard (server) doesn’t remember your previous visits.
* The badge (token) simply proves you’re allowed to be there, maybe with access to certain floors (permissions).

So yes — a **token just proves who you are and what you can do**.

---

### 2️⃣ What happens to “state”

That’s the stateless part:
the server does **not** keep per-user memory between requests.

In a **stateful** model (like old cookie sessions):

* The server had a table:
  `sessionId=abc123 → user=Edgaras, cart=3 items, lastPage=/checkout`
* It used that for context in later requests.

In a **stateless** model (tokens):

* The server doesn’t keep that table.
* Each request stands alone: the server authenticates, authorizes, and processes it from scratch using info inside (or derived from) the token.

If the server needs extra data — for example, your shopping cart — it looks it up from the **database**, not from memory tied to a session.

That’s what “stateless” really means:

> No in-memory user context between requests. Every request carries all it needs.

---

### 3️⃣ What’s inside the token, then?

Usually minimal facts:

```json
{
  "sub": "42",
  "role": "USER",
  "scope": "read:books",
  "exp": 1739678400
}
```

* `sub` → who you are
* `role` / `scope` → what you can do
* `exp` → when this proof expires

So the token doesn’t tell your *story*, it tells your *authority*.

---

### 4️⃣ How the server uses it

When you send a request with your token:

1. Server verifies its **signature** → confirms it wasn’t forged.
2. Checks **expiration**.
3. Reads **claims** (`sub`, `role`, `scope`).
4. Uses that info to decide:

   * Can this user access `/api/orders/42`?
   * Is this admin-only?
   * What data should they see?

Then the app logic handles whatever you asked — e.g., reading your orders — using the user ID from the token to query the database.

---

### 5️⃣ Does the server *need* history?

Not usually.
In modern designs, **state lives in the database**, not the session.

If it needs to remember your past actions — orders, cart, last login — that’s persisted in the DB keyed by your user ID, not kept in RAM between requests.

So “stateless” doesn’t mean “no persistence.”
It means “no transient memory that ties one HTTP request to the next.”

---

### 6️⃣ Permissions — what the token “grants”

When an auth system issues a token, it embeds what that token *allows*.
Those are your **scopes** or **roles**.

Examples:

```json
"scope": "read:orders write:profile"
```

That’s like saying:

> “This badge lets you read orders and update your profile, but not delete users.”

The backend checks these scopes to enforce access control.

If you later change a user’s permissions, future tokens will reflect that — but old tokens will keep old scopes until they expire.

---

### 7️⃣ So in summary:

* **Token = identity + permissions**, not history.
* **Server = verifier**, not rememberer.
* **State (like data or progress)** lives in the **database**, not the session.
* **Stateless** means each request can stand alone — validated and processed without prior context.
* **Scopes/roles** inside the token define what you can touch.

