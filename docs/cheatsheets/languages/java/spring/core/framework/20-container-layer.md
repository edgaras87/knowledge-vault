---
title: Container Layer
date: 2025-10-21
tags: 
    - java
    - spring
    - ioc
    - dependency-injection
    - applicationcontext
    - beanfactory
    - reflection
summary: A comprehensive cheatsheet on Spring's Container Layer, covering Inversion of Control (IoC), Dependency Injection (DI), BeanFactory, ApplicationContext, bean lifecycle, and common patterns and pitfalls.
aliases:
  - Spring Container Layer
---


# ⚙️ Container Layer — IoC, Dependency Injection & Context Cheatsheet

> **Essence:**
> The **Container Layer** is the heart of Spring — the *Inversion of Control* (IoC) engine that **creates**, **wires**, **manages**, and **coordinates** all beans.
> It gives your application memory, order, and lifecycle.

---

## 🧭 1. The Big Idea: Inversion of Control (IoC)

Traditional Java:

```java
PaymentService service = new PaymentService(new AuditService());
```

Spring style:

```java
@Autowired
PaymentService service;
```

The control has been **inverted** —
Spring now decides *when* and *how* to create your objects.

---

### 🔍 What IoC Means

* You **declare** what classes exist and how they relate.
* The container **builds and connects** them at runtime.
* Your code **depends on abstractions**, not construction logic.

IoC is the philosophy; **Dependency Injection (DI)** is the mechanism that makes it happen.

---

## 🤝 2. Dependency Injection — The Wiring Mechanism

### Types of Injection

| Style           | Example                      | When to Use                          |
| --------------- | ---------------------------- | ------------------------------------ |
| **Constructor** | `public Service(Repo r)`     | Preferred — immutable, required deps |
| **Setter**      | `@Autowired setRepo(Repo r)` | Optional dependencies                |
| **Field**       | `@Autowired Repo repo;`      | Simple, less testable                |

### Injection Flow

```
1. Container identifies required dependencies.
2. Finds matching beans by type.
3. Injects via reflection (constructor, field, or setter).
4. Calls @PostConstruct or init methods.
```

---

### 🧩 Qualifiers and Primary

When multiple beans match a type:

```java
@Service("emailNotifier")
public class EmailNotifier implements Notifier {}

@Primary
@Service
public class SmsNotifier implements Notifier {}
```

```java
@Autowired
@Qualifier("emailNotifier")
private Notifier notifier;
```

Spring rules:

1. Match by type.
2. If multiple found → use `@Primary`.
3. If qualified → use that exact name.
4. If none → throw `NoSuchBeanDefinitionException`.

---

### ⚙️ Optional and Lazy Dependencies

```java
@Autowired(required = false)
private Optional<AuditService> audit;

@Autowired
@Lazy
private HeavyService heavy; // created on first use
```

---

### 🧱 Collection Injection

Spring can inject multiple beans of the same type as a list or map:

```java
@Autowired
private List<PaymentProcessor> processors;

@Autowired
private Map<String, PaymentProcessor> processorMap;
```

---

## 🧩 3. BeanFactory — The Core Container

`BeanFactory` is the **lowest-level** IoC interface — minimal and efficient.

### Core Responsibilities

* Hold **BeanDefinitions** (metadata)
* Instantiate beans via reflection
* Perform dependency injection
* Handle scopes (singleton, prototype, etc.)

```java
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("app.xml"));
MyService s = factory.getBean(MyService.class);
```

### Key Methods

| Method                      | Purpose               |
| --------------------------- | --------------------- |
| `getBean(String name)`      | Retrieve bean by name |
| `getBean(Class<T> type)`    | Retrieve by type      |
| `containsBean(String name)` | Check existence       |
| `isSingleton(String name)`  | Check scope           |
| `getType(String name)`      | Get class type        |

> **Note:** You rarely use `BeanFactory` directly — it’s the internal engine behind `ApplicationContext`.

---

## 🧭 4. ApplicationContext — The Intelligent Container

`ApplicationContext` extends `BeanFactory` and adds:

* **Lifecycle events**
* **Resource loading**
* **Environment & profiles**
* **Internationalization**
* **AOP integration**
* **Startup hooks (CommandLineRunner, ApplicationRunner)**

---

### Typical Usage

```java
ApplicationContext ctx =
        new AnnotationConfigApplicationContext(AppConfig.class);

MyService s = ctx.getBean(MyService.class);
ctx.getBean("auditService", AuditService.class);
```

---

### Context Implementations

| Implementation                       | Purpose                        |
| ------------------------------------ | ------------------------------ |
| `AnnotationConfigApplicationContext` | Java-based config              |
| `ClassPathXmlApplicationContext`     | XML config                     |
| `FileSystemXmlApplicationContext`    | From filesystem path           |
| `WebApplicationContext`              | Web-aware context (Spring MVC) |

---

### Lifecycle Sequence

```
create context
  ↓
load BeanDefinitions
  ↓
instantiate beans
  ↓
inject dependencies
  ↓
call lifecycle methods
  ↓
publish ContextRefreshedEvent
  ↓
app ready
  ↓
on shutdown → ContextClosedEvent → destroy beans
```

---

### Event Publishing

```java
@Component
public class StartupListener implements ApplicationListener<ContextRefreshedEvent> {
    public void onApplicationEvent(ContextRefreshedEvent e) {
        System.out.println("Context ready!");
    }
}
```

or

```java
@EventListener
public void handleShutdown(ContextClosedEvent e) {
    System.out.println("Context closing...");
}
```

---

### Environment & Profiles

```java
@Profile("dev")
@Bean
public DataSource devDataSource() { ... }

@Profile("prod")
@Bean
public DataSource prodDataSource() { ... }
```

Active profile:

```bash
--spring.profiles.active=dev
```

---

## ⚙️ 5. Bean Post-Processing

Post-processors modify beans before and after initialization.

```java
@Component
public class MetricsInjector implements BeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String name) {
        // before @PostConstruct
        return bean;
    }
    public Object postProcessAfterInitialization(Object bean, String name) {
        // after initialization
        return bean;
    }
}
```

Used for:

* Autowiring processing
* `@Transactional` proxies
* Caching
* Validation

---

## 🧮 6. Context Hierarchies

Large applications may use multiple contexts:

```
Parent Context (shared configs)
   ↓
Child Context (web layer, modules)
```

A child context can access parent beans but not vice versa.
Used in modular systems or Spring MVC applications.

---

## 🧠 7. Common Patterns

### Access the Context

```java
@Component
public class ContextAwareBean implements ApplicationContextAware {
    private ApplicationContext ctx;
    public void setApplicationContext(ApplicationContext c) {
        this.ctx = c;
    }
}
```

### Programmatic Bean Access

```java
@Autowired
ApplicationContext ctx;
MyService s = ctx.getBean(MyService.class);
```

### Manual Refresh

```java
AnnotationConfigApplicationContext ctx =
        new AnnotationConfigApplicationContext();
ctx.register(AppConfig.class);
ctx.refresh();
```

---

## 🧩 8. Typical Bean Lifecycle (Container Perspective)

```
register BeanDefinition
    ↓
BeanFactory creates instance
    ↓
Autowired dependencies injected
    ↓
BeanPostProcessors run
    ↓
@PostConstruct / afterPropertiesSet()
    ↓
Context publishes events
    ↓
Bean used during runtime
    ↓
@PreDestroy / destroy()
```

---

## ⚙️ 9. Common Pitfalls

| Symptom                                 | Cause                        | Fix                            |
| --------------------------------------- | ---------------------------- | ------------------------------ |
| `BeanCurrentlyInCreationException`      | Circular dependencies        | Use setter or lazy injection   |
| `NoSuchBeanDefinitionException`         | Missing bean                 | Check component scan           |
| `IllegalStateException: context closed` | Using context after shutdown | Manage lifecycle properly      |
| `@Autowired` not working                | Class not scanned            | Adjust `@ComponentScan`        |
| Multiple beans found                    | Ambiguous type               | Use `@Qualifier` or `@Primary` |

---

## 🧭 10. Container Class Hierarchy Summary

```
BeanFactory
  ↑
ListableBeanFactory
  ↑
HierarchicalBeanFactory
  ↑
ApplicationContext
   ├─ AnnotationConfigApplicationContext
   ├─ ClassPathXmlApplicationContext
   └─ WebApplicationContext
```

---

## 🧱 11. Quick Reference Summary

| Concept              | API / Annotation                     | Description                        |
| -------------------- | ------------------------------------ | ---------------------------------- |
| Inversion of Control | —                                    | Framework controls object creation |
| Dependency Injection | `@Autowired`, `@Qualifier`           | Auto-wiring of beans               |
| Core container       | `BeanFactory`                        | Low-level bean management          |
| Full container       | `ApplicationContext`                 | IoC + events + resources           |
| Bean lookup          | `getBean(Class<T>)`                  | Retrieve from context              |
| Lifecycle            | `@PostConstruct`, `@PreDestroy`      | Init/destroy hooks                 |
| Events               | `ApplicationEvent`, `@EventListener` | Pub/sub system                     |
| Profiles             | `@Profile`                           | Conditional bean loading           |

---

### 🔗 Related

* [Concept: IoC Container](../../../../concepts/frameworks/spring/core/25-ioc-container.md)
* [Concept: Context Lifecycle](../../../../concepts/frameworks/spring/core/15-context-lifecycle.md)
* [Cheatsheet: beans-layer.md](../beans/beans-layer.md)
* [Cheatsheet: aop-layer.md](../aop/aop-layer.md)

---

### 🪞 Core Takeaway

> **The Container Layer is Spring’s mind.**
> It knows what beans exist, how they depend on each other,
> and when they should live or die.
> Reflection builds them, the Container awakens them —
> making IoC the intelligence that turns plain code into a coordinated ecosystem.

