---
title: Visibility Design
date: 2025-11-16
tags: 
  - java
  - api-design
  - best-practices
  - visibility 
  
summary: Designing APIs that evolve safely without leaking internal details
aliases:
  - Java Visibility Design Best Practices
  - Java API Visibility Strategy
---

# **Visibility Design Best Practices**

*Designing APIs that evolve safely without leaking internal details*

---

## **Contents**

1. Purpose of Visibility in API Design
2. General Rule: Start Restrictive, Widen Only When Needed
3. Understanding Java Visibility Levels
4. Why “Package-Private First” Is a Powerful Default
5. How This Works in Real Software Teams
6. Case Study: A `Slug` Value Object
7. When and How to Promote Visibility Safely
8. Notes for Multi-Team and Production Environments
9. Practical Heuristics to Follow

---

# **1. Purpose of Visibility in API Design**

Visibility isn’t just about “who can call what.”
It’s about **shaping the boundaries of your codebase**.

Proper visibility:

* keeps accidental coupling low
* prevents misuse of internal logic
* protects invariants
* preserves long-term refactorability
* clarifies intent for other developers

A method’s visibility is part of its **API contract**.
Contracts should be small and intentional.

---

# **2. General Rule: Start Restrictive, Widen Only When Needed**

The safest strategy that experienced developers use:

> **Begin with the most restrictive visibility that satisfies current use-cases.
> Only increase visibility when a real, concrete requirement appears.**

Because:

* Widening visibility (`package-private → public`) is **safe** (no breakage).
* Tightening visibility (`public → package-private`) is **breaking**, and often impossible if others depend on it.

Starting restrictive minimizes accidental public surface area.

---

# **3. Understanding Java Visibility Levels**

A quick reference, tied to intent:

| Modifier                        | Who Can Access            | Typical Use Case                                |
| ------------------------------- | ------------------------- | ----------------------------------------------- |
| `private`                       | Same class                | Internal helpers, invariants                    |
| *package-private* (no modifier) | Same package              | Module-internal API                             |
| `protected`                     | Same package + subclasses | Inheritance (rarely relevant for value objects) |
| `public`                        | Everywhere                | Stable contract exposed to rest of codebase     |

For domain value objects, inheritance isn’t a concern, so **protected usually has no role**.

---

# **4. Why “Package-Private First” Is a Powerful Default**

Package-private is a quiet superstar:

* controllers, services, infra outside the package **cannot** misuse internals
* within the domain package, internal building blocks are accessible
* you can refactor freely as long as callers stay inside the package
* promotion to `public` later is smooth and non-breaking

This keeps your module’s API **tight**, while allowing flexibility internally.

---

# **5. How This Works in Real Software Teams**

In a real multi-team setup:

* Domain packages act as “mini-modules.”
* Package-private methods are **internal** to that module.
* Public methods are **official API**—other teams will treat them as stable.
* Once something becomes public, it’s hard to remove.

This approach prevents the situation where an internal helper accidentally becomes a de-facto public contract simply because it was `public`.

---

# **6. Case Study: A `Slug` Value Object**

Imagine a domain value object:

```java
// domain/category/Slug.java
package com.example.shop.domain.category;

public record Slug(String value) {
    // constructor enforces invariants

    // main factory: for user-facing input
    public static Slug fromName(String name) { ... }

    // internal factory: for canonical stored slugs
    static Slug fromCanonical(String persisted) {
        return new Slug(persisted);
    }
}
```

### Why this is good:

**`fromName` is public**
→ the safe, normal way controllers/services should create slugs
→ it does normalization, lowercasing, cleanup

**`fromCanonical` is package-private**
→ reserved for persistence/migrations
→ most callers can’t accidentally use it
→ if later a real external need appears, visibility can be widened safely

This design makes intention clear:
one is the “normal path”, one is the “low-level path”.

---

# **7. When and How to Promote Visibility Safely**

Promoting a method from package-private to public is:

* **binary-safe**
* **source-safe**
* **deployment-safe**

It doesn’t break existing compiled code.

However:

* You are *expanding* your API surface.
* Other teams may start relying on it.
* Future refactoring becomes harder.
* You lose the freedom to change/remove it without coordination.

So the rule:

> Promote visibility only when you understand the long-term cost of making it public.

---

# **8. Notes for Multi-Team / Production Environments**

In a production environment:

* Changing to public is safe technically.
* The real cost is **ownership, expectations, and maintenance**.
* People will treat public APIs as stable contracts.

Teams typically follow guidelines like:

* “Nothing is public without a documented reason.”
* “Internal helpers remain package-private.”
* “Promotion to public requires code review + justification.”

It’s not bureaucracy — it’s protection against accidental API bloat.

---

# **9. Practical Heuristics**

Here are the rules senior devs actually follow:

**Rule 1 — Restrict now, open later.**
You’ll never regret starting with package-private.

**Rule 2 — Name APIs loudly.**
`fromName` vs `fromCanonical` signals intent without reading docs.

**Rule 3 — Public means “I support this forever.”**
Make something public only if you’re ready for long-term ownership.

**Rule 4 — If only persistence needs it, keep it internal.**
Domain value objects rarely need public low-level factories.

**Rule 5 — Refactor callers before widening visibility.**
Sometimes the caller belongs in the same package, not the method.

---

This concept—visibility-driven API design—is one of those things that silently separates careful engineering from uncontrolled growth. Once you internalize it, the shape of your code naturally becomes cleaner and more maintainable.


