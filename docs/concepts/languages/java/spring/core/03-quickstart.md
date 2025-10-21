---
title: Quickstart to Spring Core
date: 2025-10-21
tags: 
    - java
    - spring
    - ioc
    - reflection
    - architecture
summary: A gentle yet comprehensive introduction to the core concepts of Spring Framework, focusing on how reflection, dependency injection, and inversion of control work together to create a managed application environment.
aliases:
  - Spring Core Quickstart
---



# 🌿 Spring Core — A Gentle but Complete Story

> **Purpose:**
> This isn’t a list of annotations.
> It’s a story about how Java evolved from “do everything yourself” to “let the framework orchestrate the show.”
> Spring sits in that shift — teaching code to manage itself.

---

## 1. The Old Days — Manual Wiring

In plain Java, you were the conductor of a one-person orchestra.

```java
UserService user = new UserService();
PaymentService pay = new PaymentService(user);
OrderService order = new OrderService(pay);
```

You wrote `new` everywhere. You knew every connection by hand.

That works until your system grows and suddenly:

* Changing one class breaks five others.
* You can’t test one piece without booting everything.
* You have no central place that knows *what depends on what*.

Developers started dreaming of a **container** — something that could build and connect objects automatically.

That dream became **Spring**.

---

## 2. Reflection — Letting Code Look at Code

Java has a secret mirror called **reflection**.
It lets a program peek into its own classes while running:

* “What fields does this class have?”
* “Does it have an annotation called `@Component`?”
* “Can I call this method without knowing its name at compile time?”

Spring uses that mirror to **discover and build** your objects.

When you run your app, it quietly scans every class on your classpath.
When it finds one marked like this:

```java
@Component
public class PaymentService {}
```

It says: *Ah, that’s mine to manage.*
Then it creates an instance reflectively — without you writing `new PaymentService()`.

That’s where the magic begins: **the framework starts taking responsibility for creation**.

---

## 3. Beans — Living Java Objects

Spring calls these managed objects **beans**.
They’re still just Java objects, but with a difference — Spring **controls their entire life**.

A bean is born when the container starts.
It’s injected with whatever it needs.
It lives inside a structure called the **ApplicationContext**, and when the application shuts down, Spring cleans it up.

You can think of the ApplicationContext as **a living ecosystem of beans**:

```
ApplicationContext
 ├─ auditService
 ├─ paymentService
 └─ userService
```

Each one can depend on the others — but they never create each other directly.
They trust Spring to connect them.

---

## 4. Dependency Injection — Letting Someone Else Plug the Cables

In normal Java, a class builds its own tools.
With Spring, it simply says what it *needs*, and the container provides it.
That pattern is called **Dependency Injection** (DI).

Imagine a workshop:

* Without Spring: every worker buys their own hammer.
* With Spring: there’s a tool manager handing out hammers as needed.

In code:

```java
@Service
public class PaymentService {
    private final AuditService audit;

    public PaymentService(AuditService audit) {
        this.audit = audit;
    }
}
```

You don’t decide which `AuditService` to use. Spring does.
At startup, it looks at your constructor and says: *“I have an `AuditService` bean — I’ll plug it in.”*

That’s constructor injection — the cleanest and safest style.
Spring can also inject dependencies through setters or directly into fields, but constructor injection makes dependencies explicit, like stating ingredients on a recipe.

---

## 5. Inversion of Control — The Big Flip

Before Spring, you wrote code like this:

```java
PaymentService pay = new PaymentService(new AuditService());
```

You were in charge of **when and how** things were created.

Spring flips that — hence the phrase **Inversion of Control (IoC)**.
Now, you declare what your classes need, and the framework decides **when to create, connect, and destroy them**.

You stopped being the factory.
You became the designer — telling the system what components exist and how they should relate.
Spring became the **conductor** of your orchestra.

---

## 6. The ApplicationContext — Spring’s Brain

The **ApplicationContext** is Spring’s central nervous system.
When you run a Spring Boot app with:

```java
SpringApplication.run(App.class, args);
```

you’re really saying: *“Hey Spring, build my world.”*

Here’s roughly what happens:

1. Spring creates the ApplicationContext.
2. It scans your classpath for components (`@Component`, `@Service`, `@Repository`…).
3. It builds a list of **BeanDefinitions** — recipes describing how to create each bean.
4. It uses reflection to instantiate them.
5. It injects dependencies.
6. It calls initialization hooks (`@PostConstruct`).
7. Finally, it announces: “Context refreshed — everything’s alive.”

From this point on, you can request any bean, and Spring will hand you a fully wired instance.

---

## 7. Bean Lifecycle — The Circle of (Application) Life

Every bean in Spring goes through predictable stages:

```
1. Definition  → Spring learns about the class
2. Instantiation  → Object is created reflectively
3. Dependency Injection  → Fields/constructors are filled
4. Initialization  → @PostConstruct runs
5. Ready State  → Bean is active and serving
6. Destruction  → @PreDestroy runs at shutdown
```

This is why your app can start, serve, and shut down gracefully.
Spring takes care of object lifespans so you can focus on behavior.

---

## 8. Why This Architecture Matters

This container-based system unlocks a few superpowers:

* **Loose coupling** — classes don’t depend on each other directly.
* **Configurable behavior** — you can change how things work via annotations or properties.
* **Automatic lifecycle management** — no manual setup or cleanup.
* **Testability** — you can inject mocks for testing instead of real beans.

Once you experience it, going back to plain `new` feels like riding a horse after owning a spaceship.

---

## 9. AOP — Teaching Beans New Tricks

When Spring controls your objects, it can **extend their behavior**.
That’s where **Aspect-Oriented Programming (AOP)** comes in.

Think of AOP as a transparent wrapper around your beans.
When someone calls a method, Spring can sneak in extra logic before or after it — like logging, transactions, or caching — without changing the original class.

```java
@Service
public class PaymentService {
    @Transactional
    public void pay() { ... }
}
```

When you call `pay()`, you’re actually calling a **proxy**, not the raw object.
That proxy:

1. Opens a transaction,
2. Calls your method,
3. Commits or rolls back depending on the result.

Same story for `@Async`, `@Cacheable`, or `@Secured` — all powered by this wrapping trick.
That’s how crosscutting concerns stay separate from business logic.

---

## 10. How Spring Pulls It Off

Behind the scenes:

* **Reflection** lets Spring look at classes and create them dynamically.
* **IoC** gives Spring control over who creates what.
* **AOP** gives Spring control over how methods behave when called.

Together, they form a vertical architecture:

```
Reflection → builds beans
     ↓
IoC → manages relationships and lifecycles
     ↓
AOP → adds behavior at runtime
```

Each layer adds a dimension:
first existence, then orchestration, then influence.

---

## 11. The Philosophy in One Line

> **Spring’s goal is to let developers think about “what should happen,” not “how to glue it all together.”**

It doesn’t replace Java — it *elevates* it, turning plain classes into cooperative citizens inside a managed world.

---

## 12. The Lifecycle of a Spring Application

1. You press **Run**.
2. Spring starts and scans your classes.
3. It builds a map of all components.
4. It reflectively creates and connects them.
5. It wraps some in proxies for AOP.
6. It publishes “Context Refreshed” — your app is alive.
7. When you stop, it calls cleanup hooks and shuts down gracefully.

That’s the entire story — an ecosystem built from reflection, orchestration, and trust.

---

## 13. TL;DR Summary

| Concept                        | What It Means                   | Why It Matters                           |
| ------------------------------ | ------------------------------- | ---------------------------------------- |
| **Reflection**                 | Java looking at itself          | Lets Spring find and create your beans   |
| **Bean**                       | A managed Java object           | Gives you lifecycle control              |
| **Dependency Injection**       | Framework provides dependencies | Removes manual wiring                    |
| **Inversion of Control (IoC)** | Framework owns object creation  | Makes systems modular                    |
| **ApplicationContext**         | Container managing beans        | Central brain of your app                |
| **AOP / Proxies**              | Logic woven around methods      | Adds features like transactions, caching |

---

### 🪞 Final Thought

Spring isn’t a library you call — it’s an environment you live in.
It doesn’t just execute code; it **hosts** it.
Once you understand that, every `@Autowired`, every `@Transactional`, every line of your app fits into one graceful rhythm:
**discover, build, wire, run, extend, and clean up.**

