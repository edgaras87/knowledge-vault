---
title: Beans Layer  
date: 2025-10-21
tags: 
    - java
    - spring
    - bean
    - lifecycle
    - reflection
summary: A cheatsheet covering the anatomy, annotations, scopes, and lifecycle of Spring Beans, including how they are defined, created, and managed within the ApplicationContext.
aliases:
  - Spring Beans Layer
---

# 🌿 Beans Layer — Anatomy, Annotations & Lifecycle Cheatsheet

> **Essence:**
> Beans are the **living objects** Spring creates, wires, and manages inside the ApplicationContext.
> The Beans Layer covers *what* they are, *how* they live and die, and *how* their metadata is described through BeanDefinitions.

---

## 🧭 1. Concept Overview

A **Spring bean** is a normal Java object, but:

* Its lifecycle is controlled by the **Spring container**, not you.
* It’s created via reflection, wired via dependency injection, and destroyed gracefully.
* Everything about it — class, scope, dependencies, and init/destroy callbacks — is stored in a **BeanDefinition**.

Think of each bean as a *citizen* of Spring’s managed world — born, connected, and retired by the container.

---

## 🧩 2. Declaring Beans

You can register beans in three main ways:

### 🧱 Component Scanning (Annotation-based)

```java
@Component
public class PaymentService {}
```

Spring scans packages for these stereotypes:

| Annotation       | Role                                                |
| ---------------- | --------------------------------------------------- |
| `@Component`     | Generic Spring-managed bean                         |
| `@Service`       | Service-layer bean (semantic marker)                |
| `@Repository`    | Persistence-layer bean (adds exception translation) |
| `@Controller`    | Web MVC component                                   |
| `@Configuration` | Source of explicit bean definitions                 |

---

### ⚙️ Explicit Bean Registration (Configuration Class)

```java
@Configuration
public class AppConfig {
    @Bean
    public AuditService auditService() {
        return new AuditService();
    }
}
```

Each `@Bean` method returns an object that Spring registers as a managed bean.

---

### 🧮 XML Configuration (Legacy)

```xml
<bean id="paymentService" class="com.app.PaymentService"/>
```

Old style, still supported but rarely used in modern applications.

---

## 🧬 3. BeanDefinition — The Blueprint Metadata

When Spring detects a bean, it doesn’t create it immediately.
It first builds a **BeanDefinition** — a metadata record describing:

| Property            | Meaning                                |
| ------------------- | -------------------------------------- |
| `beanClass`         | Actual `Class<?>`                      |
| `scope`             | `singleton`, `prototype`, etc.         |
| `autowireMode`      | How dependencies are injected          |
| `initMethodName`    | Custom initialization method           |
| `destroyMethodName` | Cleanup method                         |
| `lazyInit`          | Whether bean is created on demand      |
| `dependsOn`         | Other beans that must be created first |

Example (simplified pseudocode):

```java
BeanDefinition bd = new BeanDefinition();
bd.setBeanClass(PaymentService.class);
bd.setScope("singleton");
bd.setInitMethodName("init");
bd.setDestroyMethodName("cleanup");
```

This blueprint is what the **BeanFactory** later uses to actually create the instance.

---

## ⚙️ 4. Bean Scopes — Controlling Lifespan

| Scope         | Meaning                    | Created              | Destroyed          |
| ------------- | -------------------------- | -------------------- | ------------------ |
| `singleton`   | One instance per container | On context start     | On context close   |
| `prototype`   | New instance per request   | On demand            | Not tracked        |
| `request`     | One per HTTP request       | On web request start | On web request end |
| `session`     | One per HTTP session       | On session start     | On session end     |
| `application` | One per ServletContext     | On context init      | On shutdown        |

```java
@Component
@Scope("prototype")
public class Task {}
```

---

## 🧠 5. Bean Lifecycle Stages

```
1. Definition created (BeanDefinition)
2. Instance constructed (reflection)
3. Dependencies injected (@Autowired, @Value)
4. Lifecycle callbacks triggered (@PostConstruct, init-method)
5. Bean enters ready state
6. On context shutdown → @PreDestroy or destroy-method called
```

### Lifecycle Annotations and Interfaces

| Type       | Example                                 | Purpose                       |
| ---------- | --------------------------------------- | ----------------------------- |
| Annotation | `@PostConstruct`                        | After injection               |
| Annotation | `@PreDestroy`                           | Before destruction            |
| Interface  | `InitializingBean.afterPropertiesSet()` | Post-injection initialization |
| Interface  | `DisposableBean.destroy()`              | Cleanup                       |
| XML Attr   | `init-method` / `destroy-method`        | Legacy equivalent             |

---

### Example

```java
@Component
public class ConnectionPool {

    @PostConstruct
    public void init() { System.out.println("Pool ready"); }

    @PreDestroy
    public void shutdown() { System.out.println("Pool closed"); }
}
```

Output:

```
Pool ready
...
Pool closed
```

---

## 🧩 6. Dependency Injection Timing

Spring always injects dependencies **before** calling any init methods.
That’s why you can safely rely on `@Autowired` fields or constructor-injected dependencies inside your `@PostConstruct`.

```
instantiate → inject → initialize → active → destroy
```

---

## 🧮 7. Lazy and Conditional Beans

### Lazy

```java
@Component
@Lazy
public class ExpensiveBean {}
```

Created only when first requested.

### Conditional

```java
@Bean
@ConditionalOnProperty("feature.x.enabled")
public FeatureX feature() { return new FeatureX(); }
```

Used heavily in Spring Boot auto-configuration.

---

## 🧭 8. Common Pitfalls

| Symptom                                  | Cause               | Fix                                 |
| ---------------------------------------- | ------------------- | ----------------------------------- |
| `BeanCurrentlyInCreationException`       | Circular dependency | Break cycle or use `@Lazy`          |
| `NullPointerException` on injected field | Bean not scanned    | Fix `@ComponentScan` path           |
| `NoSuchBeanDefinitionException`          | No bean found       | Annotate or register bean           |
| Init/destroy not called                  | Wrong signature     | Must be `public void` no-arg method |
| Prototype beans not destroyed            | Expected behavior   | Manage manually if needed           |

---

## ⚙️ 9. Inspecting Beans at Runtime

```java
@Autowired
ApplicationContext ctx;

public void debugBeans() {
    for (String name : ctx.getBeanDefinitionNames()) {
        System.out.println(name + " → " + ctx.getType(name));
    }
}
```

---

## 🧩 10. Custom BeanPostProcessor

Advanced hook that modifies beans before/after initialization.

```java
@Component
public class LoggerInjector implements BeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String name) {
        // before @PostConstruct
        return bean;
    }
    public Object postProcessAfterInitialization(Object bean, String name) {
        // after @PostConstruct
        return bean;
    }
}
```

Used internally for:

* Autowiring processor
* AOP proxy creation
* Validation, caching, and more

---

## 🧭 11. Quick Reference Summary

| Concept              | API / Annotation                          | Purpose                  |
| -------------------- | ----------------------------------------- | ------------------------ |
| Declare bean         | `@Component`, `@Service`, `@Bean`         | Register with container  |
| Describe bean        | `BeanDefinition`                          | Internal metadata        |
| Define scope         | `@Scope("...")`                           | Control lifespan         |
| Init phase           | `@PostConstruct` / `afterPropertiesSet()` | Setup                    |
| Destroy phase        | `@PreDestroy` / `destroy()`               | Cleanup                  |
| Post-processing      | `BeanPostProcessor`                       | Modify beans             |
| Lazy loading         | `@Lazy`                                   | Create on demand         |
| Conditional creation | `@Conditional...`                         | Context-based activation |

---

## 🧩 12. How BeanDefinition Links to Lifecycle

```
@Component → Scanned by Reflection
     ↓
BeanDefinition → Metadata stored
     ↓
BeanFactory → Instantiates using reflection
     ↓
@Autowire / @Value → Dependencies injected
     ↓
@PostConstruct → Init logic
     ↓
Context Active
     ↓
@PreDestroy → Cleanup
```

BeanDefinition is the **blueprint**; the Bean instance is the **living object** built from it.

---

### 🔗 Related

* [Concept: Bean Anatomy](../../../../concepts/frameworks/spring/core/10-bean-anatomy.md)
* [Concept: Context Lifecycle](../../../../concepts/frameworks/spring/core/15-context-lifecycle.md)
* [Cheatsheet: reflection-layer.md](../reflection/reflection-layer.md)
* [Cheatsheet: applicationcontext.md](../container/applicationcontext.md)

---

### 🪞 Core Takeaway

> **Reflection lets Spring build objects.
> The Beans Layer teaches them how to live.**
> Every bean in the container is born from a BeanDefinition,
> configured through annotations, and guided through a full lifecycle —
> from reflective creation to graceful destruction.

