---
title: Spring Context Lifecycle
date: 2025-10-20
tags: 
    - java
    - spring
    - applicationcontext
    - lifecycle
    - reflection
summary: A comprehensive guide to understanding the lifecycle of Spring's ApplicationContext and beans, from startup through shutdown, highlighting the role of reflection in the process.
aliases:
  - Spring Context Lifecycle
---



# 🌿 Spring Context Lifecycle — From Startup to Shutdown

### Mental model

Spring is like a **living organism**:
the ApplicationContext is its brain, and your beans are the organs.
At startup, Spring **builds**, **injects**, **activates**, and **eventually disposes** every bean in a predictable lifecycle — all orchestrated with reflection.

---

## 🧭 High-level flow

```
Spring Boot starts →
  creates ApplicationContext →
    scans classpath →
      registers beans →
        instantiates beans →
          injects dependencies →
            calls lifecycle callbacks →
              ready to serve
```

At shutdown, Spring reverses the flow:

> “Call destroy methods → close context → free resources.”

---

## 🧩 Key actors in the lifecycle

| Role                   | Description                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------- |
| **ApplicationContext** | The central Spring container (brain). Manages bean creation, wiring, events, and lifecycle. |
| **BeanDefinition**     | Metadata about each bean (class, scope, dependencies, lifecycle methods).                   |
| **BeanFactory**        | Core interface that actually creates and manages beans (ApplicationContext extends it).     |
| **BeanPostProcessor**  | Hooks to modify beans *after* instantiation but *before* use.                               |
| **Environment**        | Provides configuration properties.                                                          |
| **EventPublisher**     | Broadcasts application events (`ContextRefreshedEvent`, etc.).                              |

---

## ⚙️ Step-by-step startup sequence

### 1️⃣ Bootstrapping

Spring Boot’s `SpringApplication.run()`:

1. Creates an **ApplicationContext** (usually `AnnotationConfigApplicationContext`).
2. Prepares the **Environment** (properties, profiles).
3. Starts component scanning.

---

### 2️⃣ Scanning & registration

* Spring scans the classpath for `@Component`, `@Service`, `@Repository`, `@Controller`, and `@Configuration`.
* For each, it builds a `BeanDefinition`:

  * Bean class (`Class<?>`)
  * Scope (`singleton`, `prototype`, etc.)
  * Dependencies (`@Autowired`, `@Value`)
  * Lifecycle callbacks (`@PostConstruct`, `@PreDestroy`)

No beans are created yet — only **registered** in the context.

---

### 3️⃣ Instantiation & dependency injection

* Spring goes through all registered beans.
* It creates instances reflectively using `getDeclaredConstructor().newInstance()`.
* Then performs dependency injection:

  * **Constructor injection:** calls constructor with dependencies.
  * **Field injection:** sets fields via reflection (`Field.setAccessible(true)`).
  * **Setter injection:** invokes setter methods reflectively.

After wiring, the bean is now *constructed but not yet active*.

---

### 4️⃣ Post-processing

Now, `BeanPostProcessor`s kick in — these are internal hooks that let Spring modify beans before and after initialization.

For example:

* `AutowiredAnnotationBeanPostProcessor` → performs `@Autowired` injection.
* `CommonAnnotationBeanPostProcessor` → handles `@PostConstruct` and `@PreDestroy`.
* `AopProxyCreator` → wraps beans in proxies for `@Transactional`, `@Async`, etc.

Custom post-processors can also modify beans dynamically.

---

### 5️⃣ Initialization phase

If a bean implements certain interfaces or annotations, Spring calls them in order:

| Mechanism                               | Example           | Purpose                         |
| --------------------------------------- | ----------------- | ------------------------------- |
| `InitializingBean.afterPropertiesSet()` | custom init logic | Runs after dependency injection |
| `@PostConstruct`                        | annotated method  | Same purpose, modern style      |
| XML attribute `init-method`             | (legacy)          | Run after properties set        |

At this point, the bean is **fully initialized** and enters the live context.

---

### 6️⃣ Application ready phase

When all beans are initialized:

* `ApplicationContext` publishes `ContextRefreshedEvent`.
* `@EventListener` and `ApplicationListener` beans react.
* Spring Boot runs `CommandLineRunner` and `ApplicationRunner` beans.
* The app begins serving requests (for web apps, the embedded server starts).

---

## 🧩 Visual map

```
1. Scan classes (@Component)
2. Register BeanDefinitions
3. Instantiate via reflection
4. Inject @Autowired dependencies
5. Call @PostConstruct / init-method
6. ContextRefreshedEvent fired
7. Beans fully active
```

```
Shutdown:
1. Publish ContextClosedEvent
2. Call @PreDestroy / destroy-method
3. Clear caches & ClassLoaders
4. Exit JVM
```

---

## ⚙️ Shutdown & cleanup

When the JVM shuts down or you call `context.close()`:

* Spring publishes `ContextClosedEvent`.
* Calls `@PreDestroy` methods and `DisposableBean.destroy()`.
* Destroys singletons and releases resources.
* Unloads classes if ClassLoaders are modularized (in app servers).

---

## 🧩 Key lifecycle annotations and interfaces

| Type                      | Example                   | When called                 |
| ------------------------- | ------------------------- | --------------------------- |
| `@PostConstruct`          | `public void init()`      | After dependencies injected |
| `@PreDestroy`             | `public void cleanup()`   | Before context shutdown     |
| `InitializingBean`        | `afterPropertiesSet()`    | After injection, before use |
| `DisposableBean`          | `destroy()`               | On shutdown                 |
| `ApplicationContextAware` | `setApplicationContext()` | Inject context reference    |
| `@EventListener`          | `onApplicationEvent()`    | On published events         |

---

## 🧠 Reflection behind the scenes

Reflection drives every lifecycle stage:

* Classpath scanning (`Class.forName`)
* Bean creation (`Class<?>.newInstance()`)
* Dependency wiring (`Field.setAccessible(true)`)
* Lifecycle methods (`Method.invoke`)
* Event publishing (`find annotated listener → invoke`)

But once beans are ready, normal Java calls take over — reflection happens mostly *during startup*.

---

## ⚡ Lifecycle timings (approximate)

| Phase           | Frequency     | Reflection heavy?           |
| --------------- | ------------- | --------------------------- |
| Scanning        | Once          | ✅ Yes                       |
| Instantiation   | Once per bean | ✅ Yes                       |
| Injection       | Once per bean | ✅ Yes                       |
| Post-processing | Once per bean | ✅ Moderate                  |
| Ready state     | Continuous    | ❌ No                        |
| Shutdown        | Once          | ✅ Yes (for destroy methods) |

---

## 🧩 Simplified pseudo flow

```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

for (String name : ctx.getBeanDefinitionNames()) {
    BeanDefinition def = ctx.getBeanFactory().getBeanDefinition(name);
    Object bean = ctx.getBean(name); // triggers reflection creation/injection
}
ctx.close(); // triggers destroy phase
```

---

## 🧩 Common pitfalls

| Symptom                                  | Likely cause                         | Fix                                         |
| ---------------------------------------- | ------------------------------------ | ------------------------------------------- |
| `BeanCurrentlyInCreationException`       | Circular dependencies                | Refactor to setter or constructor injection |
| `NullPointerException` on injected field | Bean not in component scan path      | Check `@ComponentScan`                      |
| `IllegalStateException: Context closed`  | Accessing context after shutdown     | Manage lifecycle correctly                  |
| `@PostConstruct` not called              | Method not public or wrong signature | Must be `public void` with no args          |

---

## 🧩 Useful debugging hooks

```java
@SpringBootApplication
public class DebugApp implements CommandLineRunner {
    @Autowired ApplicationContext ctx;
    public void run(String... args) {
        Arrays.stream(ctx.getBeanDefinitionNames())
              .forEach(System.out::println);
    }
}
```

To trace lifecycle events:

```java
@Component
public class EventsTracer implements ApplicationListener<ApplicationEvent> {
    public void onApplicationEvent(ApplicationEvent e) {
        System.out.println("Event → " + e.getClass().getSimpleName());
    }
}
```

---

## 🧩 TL;DR Summary

| Stage           | Description                  | Reflection role             |
| --------------- | ---------------------------- | --------------------------- |
| **Scan**        | Find classes via annotations | `Class.forName`             |
| **Register**    | Store metadata               | none                        |
| **Instantiate** | Create objects               | `Constructor.newInstance()` |
| **Inject**      | Wire dependencies            | `Field.set()`               |
| **Initialize**  | Run init methods             | `Method.invoke()`           |
| **Active**      | Ready for business           | none                        |
| **Shutdown**    | Cleanup hooks                | `Method.invoke()`           |

---

### 🪞 Core takeaway

> Spring’s startup is a grand choreography of reflection — loading, wiring, and awakening beans into a self-managing ecosystem. Once ready, reflection steps offstage and your code runs as pure Java.

