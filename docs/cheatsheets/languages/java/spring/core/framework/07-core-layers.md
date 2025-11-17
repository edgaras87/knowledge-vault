---
title: Core Layers
date: 2025-10-21
tags: 
    - java
    - spring
    - core
    - architecture
    - layers
summary: A comprehensive cheatsheet detailing the five core layers of the Spring Framework, from reflection and bean management to container orchestration, AOP, and event handling, illustrating how each layer contributes to Spring's dynamic runtime capabilities.
aliases:
  - Spring Core Layers
---

# 🌱 Spring Core Layers — From Reflection to Runtime Intelligence

> **Essence:**
> Spring’s heart is its **Core Framework**, a layered architecture that turns static Java classes into a living, self-managing system.
> Each layer adds new powers: *perception → life → coordination → adaptation → communication.*

---

## 🧩 1. The Five Core Layers

```
[1] Reflection Layer   → how Spring sees and wires your classes
[2] Beans Layer        → how Spring gives those classes life
[3] Container Layer    → how Spring coordinates and manages them
[4] AOP Layer          → how Spring intercepts and enhances behavior
[5] Events Layer       → how Spring communicates and reacts to change
```

Each layer builds on the one before it — forming an upward chain from **raw Java mechanics** to **dynamic runtime orchestration**.

---

## 🪞 2. Layer 1 — Reflection Layer

**Goal:** Teach Spring how to *see* your code.

Spring begins at the lowest level of Java introspection: the **Reflection API**.
It loads your classes, inspects annotations, creates instances, and injects dependencies.

| Core Mechanisms | Description                                     |
| --------------- | ----------------------------------------------- |
| `Class<?>`      | Java’s runtime type representation              |
| Reflection API  | Read annotations, fields, constructors, methods |
| `@Autowired`    | Dependency injection using reflection           |
| ClassLoader     | Loads `.class` files into Metaspace             |

**Mental model:**
Reflection is the microscope — Spring examines your code, understands it, and prepares to animate it.

→ See: [`reflection-layer.md`](reflection/reflection-layer.md)

---

## 🌿 3. Layer 2 — Beans Layer

**Goal:** Give your classes *life*.

Spring transforms static class definitions into managed, stateful objects called **Beans**.
Each bean is born from a **BeanDefinition**, which describes its type, scope, dependencies, and lifecycle methods.

| Core Concepts                           | Description                         |
| --------------------------------------- | ----------------------------------- |
| `@Component`, `@Service`, `@Repository` | Mark beans for auto-detection       |
| `@Bean`                                 | Define explicit beans in config     |
| `@PostConstruct`, `@PreDestroy`         | Lifecycle callbacks                 |
| `BeanDefinition`                        | Metadata blueprint for each bean    |
| Scopes                                  | Singleton, prototype, request, etc. |

**Mental model:**
Beans are the organs — alive within the Spring organism, created, wired, and disposed of predictably.

→ See: [`beans-layer.md`](beans/beans-layer.md)

---

## ⚙️ 4. Layer 3 — Container Layer

**Goal:** Coordinate and manage all living beans.

At this stage, Spring evolves from a builder into a **container** —
a runtime environment that performs **Inversion of Control (IoC)** and **Dependency Injection (DI)**.

| Core Components            | Role                                                                      |
| -------------------------- | ------------------------------------------------------------------------- |
| **BeanFactory**            | The raw IoC engine that instantiates and wires beans                      |
| **ApplicationContext**     | The enhanced container: adds lifecycle events, resource loading, profiles |
| **Dependency Injection**   | Automatic connection of beans by type/name                                |
| **Profiles & Environment** | Context configuration and selection                                       |
| **Lifecycle Events**       | Signals like `ContextRefreshedEvent` and `ContextClosedEvent`             |

**Mental model:**
The Container is the brain — it knows every bean, how they depend on each other, and when to awaken or retire them.

→ See: [`container-layer.md`](container/container-layer.md)

---

## 🧠 5. Layer 4 — AOP Layer

**Goal:** Add *intelligence* — the ability to intercept and extend behavior.

AOP (Aspect-Oriented Programming) lets Spring weave extra logic into existing methods without modifying code.
It creates **proxies** that wrap your beans and inject crosscutting concerns like transactions, caching, and logging.

| Core Elements                  | Description                                               |
| ------------------------------ | --------------------------------------------------------- |
| `@Aspect`                      | Marks a class containing crosscutting logic               |
| `@Before`, `@After`, `@Around` | Advice that runs around methods                           |
| `execution(...)`               | Pointcut expressions targeting specific methods           |
| Proxies                        | JDK or CGLIB-based runtime wrappers                       |
| Built-in Aspects               | `@Transactional`, `@Async`, `@Cacheable`, `@PreAuthorize` |

**Mental model:**
AOP is the nervous system — it controls reflexes, adds patterns of behavior, and keeps your core logic pure.

→ See: [`aop-layer.md`](aop/aop-layer.md)

---

## 📡 6. Layer 5 — Events Layer

**Goal:** Enable *communication* — make Spring aware of its own changes.

Spring’s event system allows components to **broadcast** and **listen** for signals.
It uses the Observer pattern to loosely couple actions across the app.

| Core Concepts               | Description                                            |
| --------------------------- | ------------------------------------------------------ |
| `ApplicationEvent`          | Base event class                                       |
| `@EventListener`            | Declarative event listener                             |
| `ApplicationEventPublisher` | Publishes events to the container                      |
| Built-in Events             | `ContextRefreshedEvent`, `ApplicationReadyEvent`, etc. |
| Async Events                | `@Async` + `@EnableAsync` for background handling      |

**Mental model:**
Events are the heartbeat — they keep the container alive, synchronized, and reactive to its own lifecycle.

→ See: [`events-layer.md`](events/events-layer.md)

---

## 🧬 7. Full System Flow — From Class to Context

```
1. JVM loads classes into Metaspace.
2. Reflection scans annotations and builds BeanDefinitions.
3. IoC container creates beans and injects dependencies.
4. Post-processors modify or proxy beans (AOP).
5. Context publishes lifecycle events.
6. Beans operate in a managed runtime ecosystem.
7. On shutdown → destroy methods + ContextClosedEvent.
```

---

## 🧱 8. Spring Core Layer Map (Visual)

```
┌────────────────────────────────────────────┐
│            [5] Events Layer                │
│   Communicates lifecycle & app events      │
└────────────────────────────────────────────┘
                ↑
┌────────────────────────────────────────────┐
│            [4] AOP Layer                   │
│   Adds crosscutting logic via proxies      │
└────────────────────────────────────────────┘
                ↑
┌────────────────────────────────────────────┐
│            [3] Container Layer             │
│   IoC engine managing all beans            │
└────────────────────────────────────────────┘
                ↑
┌────────────────────────────────────────────┐
│            [2] Beans Layer                 │
│   Defines bean metadata & lifecycle        │
└────────────────────────────────────────────┘
                ↑
┌────────────────────────────────────────────┐
│            [1] Reflection Layer            │
│   Reflection + @Autowired wiring           │
└────────────────────────────────────────────┘
```

---

## 🪞 9. Core Takeaway

> **Spring Core isn’t just a framework — it’s a living architecture.**
>
> Each layer adds a new dimension of intelligence:
>
> * **Reflection** lets Spring *see*.
> * **Beans** let it *live*.
> * **Container** lets it *think*.
> * **AOP** lets it *act and adapt*.
> * **Events** let it *communicate*.
>
> Together, they transform ordinary Java code into a **self-managing runtime organism** — the foundation of the modern Spring ecosystem.

