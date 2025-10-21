---
title: Bean Anatomy
date: 2025-10-21
tags: 
    - java
    - spring
    - bean
    - lifecycle
    - reflection
summary: A deep dive into the anatomy of a Spring Bean, exploring its definition, scope, proxy mechanisms, and lifecycle hooks, with a focus on the role of reflection throughout the process.
aliases:
  - Spring Bean Anatomy
---

# 🧩 Spring Bean Anatomy — Definition, Scope, Proxy, and Lifecycle Hooks

> **Essence:** A *Spring Bean* is a **managed Java object** whose **creation, wiring, lifecycle, and sometimes even identity** are controlled by the Spring container — not by your code.
> It’s a living object with metadata, behavior, and reflective wrapping.

---

## 🌱 The Mental Model

A *bean* in Spring is to the container what a *cell* is to an organism:
it has a **definition (DNA)**, **a lifecycle (birth → death)**, and **a membrane (proxy)** that controls how it interacts with the outside world.

---

## 🧬 1. Bean Definition — the DNA

Before a bean exists, Spring builds a **`BeanDefinition`** — metadata describing how to create it.

```
@Component → scanned
@Bean → declared
XML → defined
```

Each definition contains:

| Field                         | Description                                     |
| ----------------------------- | ----------------------------------------------- |
| `beanClass`                   | The `Class<?>` object of the bean               |
| `scope`                       | `"singleton"`, `"prototype"`, `"request"`, etc. |
| `dependencies`                | Other beans to inject                           |
| `initMethod`, `destroyMethod` | Lifecycle callbacks                             |
| `lazyInit`                    | Whether to delay creation until needed          |
| `primary`, `qualifiers`       | Conflict resolution hints                       |

The `BeanDefinition` lives in memory long before any object is created.

---

## ⚙️ 2. Instantiation — the Birth

When the context is refreshed, Spring instantiates the bean **via reflection**:

```java
Object bean = beanClass.getDeclaredConstructor().newInstance();
```

Then it resolves all declared dependencies (`@Autowired`, constructor args, etc.).

At this point:

* The object exists.
* Dependencies are injected.
* Lifecycle callbacks haven’t yet run.

This is the *infant bean* stage.

---

## 🧩 3. Initialization — Becoming Alive

Spring then applies **post-processing and lifecycle interfaces**:

| Mechanism                               | Trigger             | Purpose                             |
| --------------------------------------- | ------------------- | ----------------------------------- |
| `@PostConstruct`                        | Annotation          | Custom setup logic                  |
| `InitializingBean.afterPropertiesSet()` | Interface           | Legacy init hook                    |
| `init-method`                           | XML config          | Alternative init                    |
| `BeanPostProcessor`                     | Global interceptors | Modify/replace beans after creation |

For example, a post-processor might wrap a bean in a **proxy** to add behavior (`@Transactional`, `@Async`, etc.).

After initialization, the bean is **fully active** — ready to be used or injected elsewhere.

---

## 🪞 4. Proxies — The Membrane

A *proxy* is a thin reflective wrapper around your bean.
It intercepts method calls to add extra logic like transactions, security, or async execution.

### Two main types:

| Type                  | Tool               | Used For         |
| --------------------- | ------------------ | ---------------- |
| **JDK Dynamic Proxy** | Java’s `Proxy` API | Interfaces only  |
| **CGLIB Proxy**       | Bytecode subclass  | Concrete classes |

Example:

```java
@Transactional
public class PaymentService { ... }
```

Spring replaces it at runtime with something like:

```
PaymentService$$EnhancerBySpringCGLIB$$abc123
```

When you call a method, the proxy runs code *before and after* your logic (start transaction → invoke target → commit/rollback).

---

## 🧭 5. Scopes — The Lifecycle Context

A bean’s *scope* defines **how long it lives** and **where it’s shared**:

| Scope         | Description                               |
| ------------- | ----------------------------------------- |
| `singleton`   | One shared instance per context (default) |
| `prototype`   | New instance every injection              |
| `request`     | One per HTTP request (web only)           |
| `session`     | One per HTTP session                      |
| `application` | One per servlet context                   |
| `websocket`   | One per WebSocket session                 |

Example:

```java
@Scope("prototype")
@Component
public class TaskProcessor { ... }
```

`@Autowired` of a prototype bean creates a *fresh* instance each time.

---

## 🧩 6. Awareness Interfaces — Talking to the Container

Some beans want to “know” about their environment.
Spring injects this information through special **Aware interfaces**:

| Interface                 | What you get                   |
| ------------------------- | ------------------------------ |
| `ApplicationContextAware` | Access to the context          |
| `BeanNameAware`           | The name assigned to your bean |
| `EnvironmentAware`        | Config and property access     |
| `ResourceLoaderAware`     | File/resource utilities        |

These aren’t common for business logic, but essential for infrastructure beans (framework internals, logging, plugin systems).

---

## 🌇 7. Destruction — Graceful Death

At shutdown or when the context closes:

| Mechanism                  | When                    |
| -------------------------- | ----------------------- |
| `@PreDestroy`              | Before bean destruction |
| `DisposableBean.destroy()` | Legacy cleanup          |
| `destroy-method`           | XML config              |
| `ContextClosedEvent`       | Published globally      |

Spring calls these hooks reflectively to release connections, threads, caches, etc.

---

## 🧩 8. The Full Lifecycle Map

```
1. Scan classes → build BeanDefinition
2. Instantiate → inject dependencies
3. Apply BeanPostProcessors
4. Call @PostConstruct / init-method
5. Bean ready for use
6. (Optionally) wrapped in proxy
7. Serve requests or logic
8. Context shutdown → call @PreDestroy
```

---

## ⚡ Putting it Together — Reflection’s Thread

Reflection appears at every layer:

* **Definition stage:** reads annotations (`Class.isAnnotationPresent`)
* **Instantiation stage:** calls constructors
* **Injection stage:** sets fields or methods
* **Initialization stage:** invokes annotated methods
* **Proxying stage:** intercepts via dynamic subclasses

Yet once the bean is active, reflection disappears — normal method calls dominate.

---

## 🧭 TL;DR Summary

| Aspect          | Description        | Driven by Reflection? |
| --------------- | ------------------ | --------------------- |
| Definition      | Metadata blueprint | ✅                     |
| Instantiation   | Create object      | ✅                     |
| Injection       | Wire dependencies  | ✅                     |
| Post-processing | Modify or proxy    | ✅                     |
| Active state    | Business logic     | ❌                     |
| Destruction     | Cleanup hooks      | ✅                     |

---

### 🪞 Core takeaway

> A Spring Bean is not just an object — it’s a **meta-object**, born from metadata, shaped by reflection, guarded by proxies, and governed by lifecycle rules. Once awakened, it becomes indistinguishable from plain Java — until the next context refresh.

---

