---
title: Spring Core Quick-Starter  
date: 2025-10-21
tags: 
    - java
    - spring
    - core
    - reflection
    - bean
    - ioc
    - aop
    - events
summary: reflection, bean anatomy, dependency injection, IoC container, application context lifecycle, AOP, and events.
aliases:
  - Spring Core Quickstart
---



# ⚙️ Spring Core Quick-Starter — All-in-One Cheatsheet

> **Essence:**
> Everything that powers Spring — from reflection and beans to IoC, lifecycle, AOP, and events — condensed for daily reference.
> Think of it as your cockpit dashboard: every switch labeled, no theory required.

---

## 🌱 1. Reflection — How Spring Sees Your Code

### Core Idea

Spring uses Java **reflection** to discover, instantiate, and wire classes at runtime.

### Java Basics

```java
Class<?> c = PaymentService.class;
Object o = c.getDeclaredConstructor().newInstance();
Field f = c.getDeclaredField("audit");
f.setAccessible(true);
f.set(o, new AuditService());
```

### In Spring

| Operation          | Reflection Use              |
| ------------------ | --------------------------- |
| Classpath scanning | `Class.forName()`           |
| Bean creation      | `Constructor.newInstance()` |
| Field injection    | `Field.set()`               |
| Lifecycle methods  | `Method.invoke()`           |

### Key Annotation

`@Autowired` — tells Spring to inject a dependency automatically.

---

## 🧩 2. Bean Anatomy — What Spring Manages

### Definition

A **bean** is any Java object **managed by Spring** — created, wired, and destroyed inside the ApplicationContext.

### Lifecycle Overview

```
instantiate → inject → initialize → active → destroy
```

### Lifecycle Hooks

| Stage          | Hook                   | Description                      |
| -------------- | ---------------------- | -------------------------------- |
| Initialization | `@PostConstruct`       | Runs after dependencies injected |
| Initialization | `afterPropertiesSet()` | From `InitializingBean`          |
| Destruction    | `@PreDestroy`          | Runs before context shutdown     |
| Destruction    | `destroy()`            | From `DisposableBean`            |

### Core Annotations

| Annotation       | Role                                       |
| ---------------- | ------------------------------------------ |
| `@Component`     | Marks a Spring bean                        |
| `@Service`       | Specialized component for service layer    |
| `@Repository`    | DAO-level component, exception translation |
| `@Configuration` | Source of `@Bean` definitions              |
| `@Bean`          | Defines method-produced beans              |

---

## 🧠 3. Dependency Injection — How Beans Connect

### Injection Styles

| Type            | Example                         | Notes                    |
| --------------- | ------------------------------- | ------------------------ |
| **Constructor** | `public Service(Repo repo)`     | Preferred — immutable    |
| **Setter**      | `@Autowired setRepo(Repo r)`    | Optional or late binding |
| **Field**       | `@Autowired private Repo repo;` | Simple, less testable    |

### Qualifiers

| Annotation               | Purpose                      |
| ------------------------ | ---------------------------- |
| `@Qualifier("beanName")` | Choose specific bean         |
| `@Primary`               | Default bean for a type      |
| `@Profile("dev")`        | Load only for active profile |

### Optional/Lazy

```java
@Autowired(required = false)
@Lazy
private ExpensiveService heavy;
```

Spring injects a proxy, creates bean only when used.

---

## 🧬 4. IoC Container — Who Controls the Show

### Philosophy

**Inversion of Control (IoC)** means *the framework controls object creation, not you.*

### Core Interfaces

| Interface            | Purpose                                              |
| -------------------- | ---------------------------------------------------- |
| `BeanFactory`        | Minimal container for DI                             |
| `ApplicationContext` | Full-featured container with events, i18n, resources |

### Common Context Classes

| Class                                | Usage             |
| ------------------------------------ | ----------------- |
| `AnnotationConfigApplicationContext` | Plain Java config |
| `ClassPathXmlApplicationContext`     | Legacy XML config |
| `SpringApplication`                  | Boot entry point  |

### Useful Methods

```java
ctx.getBean(MyService.class);
ctx.containsBean("auditService");
ctx.getEnvironment().getActiveProfiles();
ctx.close();
```

### BeanDefinition Metadata

| Property        | Meaning                               |
| --------------- | ------------------------------------- |
| `beanClass`     | Actual `Class<?>`                     |
| `scope`         | `singleton` / `prototype` / `request` |
| `initMethod`    | Name of init method                   |
| `destroyMethod` | Cleanup method                        |

---

## 🧭 5. Application Context Lifecycle — From Start to Stop

```
Spring Boot run()
  ↓
create ApplicationContext
  ↓
scan classpath
  ↓
register BeanDefinitions
  ↓
instantiate + inject
  ↓
post-process + initialize
  ↓
publish ContextRefreshedEvent
  ↓
ready to serve
  ↓
(ContextClosedEvent → destroy hooks)
```

### Key Events

| Event                   | When Triggered                |
| ----------------------- | ----------------------------- |
| `ContextRefreshedEvent` | All beans initialized         |
| `ContextStartedEvent`   | Context started (rarely used) |
| `ContextClosedEvent`    | Context shutting down         |
| `ContextStoppedEvent`   | Graceful stop                 |
| `ApplicationReadyEvent` | Boot app fully started        |

---

## ⚙️ 6. AOP — How Spring Adds Behavior Dynamically

### Core Idea

**Aspect-Oriented Programming (AOP)** lets Spring weave extra behavior (transactions, caching, logging) around method calls.

### Key Components

| Term         | Meaning                             |
| ------------ | ----------------------------------- |
| **Aspect**   | Class containing crosscutting logic |
| **Advice**   | Code to run before/after methods    |
| **Pointcut** | Rule for selecting target methods   |
| **Proxy**    | Wrapper intercepting method calls   |

### Common Annotations

| Annotation                           | Role                       |
| ------------------------------------ | -------------------------- |
| `@Aspect`                            | Declares an aspect class   |
| `@Before("execution(...)")`          | Run before method          |
| `@After("execution(...)")`           | Run after method           |
| `@Around("execution(...)")`          | Wrap method (full control) |
| `@AfterReturning` / `@AfterThrowing` | Handle success/error       |

### Proxy Types

| Type              | Mechanism                 | Applies To       |
| ----------------- | ------------------------- | ---------------- |
| JDK Dynamic Proxy | `java.lang.reflect.Proxy` | Interfaces       |
| CGLIB Proxy       | Bytecode subclass         | Concrete classes |

### Built-in Use Cases

| Feature      | Annotation       | Behavior                 |
| ------------ | ---------------- | ------------------------ |
| Transactions | `@Transactional` | Begin/commit/rollback    |
| Async        | `@Async`         | Run in background thread |
| Caching      | `@Cacheable`     | Store return values      |
| Security     | `@PreAuthorize`  | Access control checks    |

---

## 📡 7. Events — How Spring Talks Internally

### Publish & Listen

```java
@Component
public class AppReadyListener implements ApplicationListener<ApplicationReadyEvent> {
    public void onApplicationEvent(ApplicationReadyEvent e) {
        System.out.println("App started!");
    }
}
```

Or annotation-style:

```java
@EventListener
public void handle(ContextClosedEvent e) {
    System.out.println("Context closed.");
}
```

### Custom Events

```java
class OrderCreatedEvent extends ApplicationEvent { ... }

ctx.publishEvent(new OrderCreatedEvent(this));
```

---

## 🧩 8. Common Gotchas

| Problem                            | Cause                                      | Fix                                      |
| ---------------------------------- | ------------------------------------------ | ---------------------------------------- |
| `NoSuchBeanDefinitionException`    | Missing or mismatched type                 | Add `@Component` or correct `@Qualifier` |
| `BeanCurrentlyInCreationException` | Circular constructor injection             | Break cycle with setter injection        |
| `@PostConstruct` not called        | Method not `public void` / wrong signature | Fix method definition                    |
| AOP not triggered                  | Self-invocation inside same class          | Move call to another bean                |

---

## 🔍 9. Useful Debugging Tricks

List all beans:

```java
Arrays.stream(ctx.getBeanDefinitionNames()).forEach(System.out::println);
```

Trace events:

```java
@Component
public class EventsTracer implements ApplicationListener<ApplicationEvent> {
    public void onApplicationEvent(ApplicationEvent e) {
        System.out.println("Event → " + e.getClass().getSimpleName());
    }
}
```

Inspect proxy target:

```java
AopUtils.getTargetClass(bean);
```

---

## 🧬 10. Hierarchy Map

```
[Reflection Layer]
   ├─ Reflection API
   └─ @Autowired injection
        ↓
[Beans Layer]
   ├─ Bean annotations & lifecycle
   └─ BeanDefinition metadata
        ↓
[Container Layer]
   ├─ Dependency injection & ApplicationContext
   └─ BeanFactory, IoC flow
        ↓
[AOP Layer]
   ├─ Aspects & proxies
        ↓
[Events Layer]
   └─ Application events & listeners
```

---

## 🪞 Core Takeaway

> Spring turns static Java code into a living ecosystem.
> Reflection lets it see your classes,
> IoC lets it control their creation,
> AOP lets it enrich their behavior,
> and the event system lets it coordinate the whole show.

---

### 🔗 Related

* **Conceptual Overview:** [`concepts/frameworks/spring/core/quick-start.md`](../../../../concepts/frameworks/spring/core/quick-start.md)
* **Blueprint:** [`meta/blueprint.md`](../../../../meta/blueprint.md)
* **Next:** explore `beans/bean-lifecycle.md` and `container/applicationcontext.md` for deeper operational detail.

