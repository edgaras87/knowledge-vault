---
title: Dependency Injection Patterns 
date: 2025-10-21
tags: 
    - java
    - spring
    - ioc
    - di
    - reflection
summary: A comprehensive guide to Dependency Injection patterns in Spring, covering Constructor, Setter, and Field Injection, with a focus on how reflection enables these mechanisms.
aliases:
  - Spring Dependency Injection Patterns
---


# 🧩 Dependency Injection Patterns — Constructor, Setter, and Field Injection

> **Essence:**
> Dependency Injection (DI) is how Spring supplies an object’s collaborators **from the outside** — instead of the object creating them.
> It’s the *hands-on implementation* of Inversion of Control (IoC).
> You don’t “new” dependencies — the **container does**, using reflection and metadata.

---

## 🧭 Mental Model

Traditional Java:

```java
public class PaymentService {
    private final AuditService audit = new AuditService();
}
```

You control everything — creation, wiring, lifecycle.
This is **tight coupling**.

Spring-style DI:

```java
@Service
public class PaymentService {
    private final AuditService audit;
    public PaymentService(AuditService audit) { this.audit = audit; }
}
```

Now Spring decides **what instance** of `AuditService` to inject.
That’s IoC in practice — *you lose control of creation, gain flexibility and testability.*

---

## 🧬 1. Constructor Injection — *Strong, Immutable, Recommended*

> Dependencies are passed through the constructor.
> The bean is **fully initialized at creation time** — no mutable state afterward.

```java
@Service
public class PaymentService {
    private final AuditService audit;
    private final NotificationService notifier;

    @Autowired
    public PaymentService(AuditService audit, NotificationService notifier) {
        this.audit = audit;
        this.notifier = notifier;
    }
}
```

### ✅ Pros

* Enforces immutability — all dependencies known upfront.
* Easy to test (just call constructor).
* Prevents circular dependencies (Spring detects them early).
* Works cleanly with `final` fields.

### ⚠️ Cons

* Slightly verbose for many parameters.
* If there’s a circular dependency, you must refactor (Spring can’t resolve it automatically).

### 🧠 Under the hood

* Spring reads constructor metadata using reflection.
* Chooses the “primary” constructor (or one annotated with `@Autowired`).
* Instantiates dependencies recursively, then calls the constructor reflectively.

---

## 🧩 2. Setter Injection — *Flexible, Mutable, Legacy-friendly*

> Dependencies are provided after the object is created, through setter methods.

```java
@Service
public class PaymentService {
    private AuditService audit;

    @Autowired
    public void setAuditService(AuditService audit) {
        this.audit = audit;
    }
}
```

### ✅ Pros

* Useful for optional dependencies (you can omit some setters).
* Plays nicely with legacy beans that have no constructor injection.
* Can rewire or modify dependency after creation (in rare cases).

### ⚠️ Cons

* Mutable → less safe.
* Object might exist in half-initialized state if not all setters are called.
* Harder to enforce required dependencies.

### 🧠 Reflection flow

1. Spring creates instance via no-arg constructor.
2. Finds all methods annotated with `@Autowired`.
3. Invokes them reflectively with matching beans.

---

## 🪞 3. Field Injection — *Shortcut, but Dangerous in Core Code*

> Dependencies are injected directly into private fields using reflection.
> It’s concise, but hides dependencies and complicates testing.

```java
@Service
public class PaymentService {
    @Autowired
    private AuditService audit;
}
```

### ✅ Pros

* Minimal boilerplate.
* Handy for prototypes, quick demos, or tests.

### ⚠️ Cons

* Hidden dependencies — constructor doesn’t reveal requirements.
* Harder to write unit tests (can’t mock easily without reflection tools).
* No immutability guarantees.
* Can fail silently if `@Autowired(required = false)` is used carelessly.

### 🧠 Reflection flow

* Spring sets field accessibility with `setAccessible(true)` and assigns value directly:

  ```java
  field.set(beanInstance, dependencyInstance);
  ```
* No constructor or setter involved.

---

## 🧩 4. Qualifiers & Ambiguity Resolution

When multiple beans of the same type exist, Spring must decide which one to inject.
That’s where **qualifiers** come in.

```java
@Service("emailNotifier")
public class EmailNotificationService implements Notifier {}

@Service("smsNotifier")
public class SmsNotificationService implements Notifier {}

@Service
public class PaymentService {
    private final Notifier notifier;
    public PaymentService(@Qualifier("emailNotifier") Notifier notifier) {
        this.notifier = notifier;
    }
}
```

Other options:

* `@Primary` — marks default bean for a type.
* `@Qualifier` — explicitly selects one.
* `@Profile` — activates beans per environment profile.

---

## ⚙️ 5. Optional and Lazy Injection

| Technique                      | Description                                         |
| ------------------------------ | --------------------------------------------------- |
| `@Autowired(required = false)` | Allows optional dependency; null if absent.         |
| `Optional<T>`                  | Preferred over `required=false`, more explicit.     |
| `@Lazy`                        | Injects a proxy; bean created only when first used. |

```java
@Autowired
@Lazy
private ExpensiveService heavy;
```

At runtime, Spring injects a **proxy**, not the real bean.
The target is created only when `heavy` is first called.

---

## 🧩 6. When to Use Which

| Pattern         | Ideal Use Case                              | Notes                                 |
| --------------- | ------------------------------------------- | ------------------------------------- |
| **Constructor** | Mandatory dependencies; core business beans | Clean, testable, immutable            |
| **Setter**      | Optional dependencies; legacy beans         | Useful when partial config is allowed |
| **Field**       | Quick wiring; small demos or tests          | Avoid in production core logic        |

---

## 🧠 7. How Reflection Powers All DI

Regardless of style, Spring always:

1. Retrieves the target `Class<?>`.
2. Scans for `@Autowired`, `@Qualifier`, etc.
3. Resolves bean dependencies recursively.
4. Uses reflection to call:

   * Constructor → `newInstance(args)`
   * Setter → `method.invoke(bean, dep)`
   * Field → `field.set(bean, dep)`

That’s how the DI patterns are unified internally — they’re just *different reflection entry points*.

---

## 🧭 8. Common Pitfalls

| Symptom                                  | Root Cause                      | Fix                                      |
| ---------------------------------------- | ------------------------------- | ---------------------------------------- |
| `NoSuchBeanDefinitionException`          | Dependency type not found       | Add or scan the missing bean             |
| `BeanCurrentlyInCreationException`       | Circular constructor injection  | Refactor one side to setter injection    |
| `NullPointerException` on injected field | Bean not in component scan path | Adjust `@ComponentScan` or configuration |
| Conflicting beans                        | Multiple candidates             | Add `@Qualifier` or `@Primary`           |

---

## 🧩 9. Conceptual Summary

| Layer              | What happens          | Reflection role           |
| ------------------ | --------------------- | ------------------------- |
| **BeanDefinition** | Dependencies recorded | Reads annotations         |
| **Instantiation**  | Bean created          | Calls constructor         |
| **Injection**      | Dependencies wired    | Sets fields/methods       |
| **Initialization** | Post-construct logic  | Invokes annotated methods |

---

## 🪞 TL;DR

> All DI patterns are just different “doors” through which Spring enters your class.
> Constructor injection is the front door — safest and most explicit.
> Setter is the side door — flexible, but open to misuse.
> Field is the window — quick, but risky if overused.

### 🧩 Core Takeaway

> **Dependency Injection** is *how* Spring performs **Inversion of Control**.
> It’s not magic — just reflection, metadata, and a disciplined refusal to use `new`.
