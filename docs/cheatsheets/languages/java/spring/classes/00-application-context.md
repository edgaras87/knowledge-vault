---
title: ApplicationContext
date: 2025-10-20
tags:
    - java
    - spring
    - applicationcontext
    - reflection
summary: A cheatsheet explaining Spring's ApplicationContext, how it uses configuration classes and reflection to manage beans at runtime.
aliases:
  - ApplicationContext
---



# 🌿 Spring ApplicationContext — The Heart of Spring

---

### 🧠 What it really is

The **ApplicationContext** is the **core container** of Spring —
the part that knows about every bean in your project:
how to create it, when to wire it, and when to destroy it.

If your application were a city, the ApplicationContext would be the **city hall** — it keeps the registry of all citizens (beans), their relationships, and their lifecycle rules.

---

## 🪞 The mental model

```
@Configuration class → metadata
AnnotationConfigApplicationContext → container
Beans → live objects managed by the container
```

In practice:

```java
AnnotationConfigApplicationContext ctx =
        new AnnotationConfigApplicationContext(AppConfig.class);
```

That single line **starts Spring’s world**.

---

### 🧩 Step 1: You hand Spring your blueprint

`AppConfig.class` is your **blueprint** — the configuration that tells Spring:

> “Here’s what classes exist, and here’s how to make them.”

It might define beans directly:

```java
@Configuration
public class AppConfig {
    @Bean
    public PaymentService paymentService() {
        return new PaymentService(auditService());
    }

    @Bean
    public AuditService auditService() {
        return new AuditService();
    }
}
```

Or it might tell Spring where to look:

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {}
```

Either way, **you’re not giving Spring the beans** —
you’re giving it **the map** to find or create them later.

---

### 🧩 Step 2: Spring reads and registers

`AnnotationConfigApplicationContext` reads the annotations on `AppConfig`:

* `@Configuration` → this class defines beans.
* `@Bean` → register this method’s return type as a bean.
* `@ComponentScan` → search for annotated components (`@Service`, `@Controller`, etc.).

Spring doesn’t build anything yet — it just **collects metadata** and builds an internal *BeanDefinition registry* (a big plan for what to create).

---

### 🧩 Step 3: You request beans

When you ask:

```java
Object bean = ctx.getBean("paymentService");
```

Spring checks if that bean already exists.
If not, it:

1. Looks up its **BeanDefinition**.
2. Uses **reflection** to create the object.
3. Injects dependencies (also via reflection).
4. Runs any init methods (`@PostConstruct`, `afterPropertiesSet()`).

After that, the bean becomes part of the live **ApplicationContext**.

---

### 🧩 Step 4: Context lifecycle

```java
ctx.close();
```

When the context closes:

* Spring calls all `@PreDestroy` or `destroy()` methods.
* Releases resources (DB pools, thread pools, etc.).
* Clears internal caches.

That’s the end of the bean lifecycle.

---

## 🔍 Under the hood

When you run:

```java
AnnotationConfigApplicationContext ctx =
        new AnnotationConfigApplicationContext(AppConfig.class);
```

Spring does roughly this:

```java
// Pseudo code
AppConfig metadata = readAnnotations(AppConfig.class);
List<BeanDefinition> beans = scanPackages(metadata);

for (BeanDefinition def : beans) {
    Class<?> type = def.getBeanClass();
    Object instance = type.getDeclaredConstructor().newInstance(); // reflection
    injectDependencies(instance);
    storeInContext(def.getName(), instance);
}
```

Spring uses **reflection** to:

* Load classes (`Class.forName()`)
* Create instances (`newInstance()`)
* Inject fields and methods (`setAccessible(true)` → `field.set()`)
* Call initialization methods

---

## 🧩 Relationship between `AppConfig` and `AnnotationConfigApplicationContext`

| Concept                              | Role                                                                      | Analogy         |
| ------------------------------------ | ------------------------------------------------------------------------- | --------------- |
| `AppConfig`                          | Blueprint that *describes* beans (metadata).                              | Recipe book     |
| `AnnotationConfigApplicationContext` | The engine that *reads the blueprint* and builds beans.                   | Chef            |
| Bean                                 | Actual instance built from metadata.                                      | The dish        |
| Reflection                           | The way the chef uses the recipe (reads annotations, calls constructors). | Cooking process |

So yes — your earlier understanding was right:

> AppConfig holds metadata. The ApplicationContext reads it and builds the beans dynamically.

---

## 🧩 Visual summary

```
AppConfig.class  (metadata)
        ↓
AnnotationConfigApplicationContext
        ↓  reads annotations via reflection
BeanDefinitions (blueprints)
        ↓  reflectively instantiate + inject
Live Beans in ApplicationContext
```

---

## ✅ TL;DR Summary

| Concept                | Description                                                          |
| ---------------------- | -------------------------------------------------------------------- |
| **ApplicationContext** | The Spring container that manages all beans.                         |
| **AppConfig**          | Blueprint class defining or locating beans.                          |
| **Annotations**        | Metadata for Spring to interpret.                                    |
| **Reflection**         | The mechanism Spring uses to create and wire beans dynamically.      |
| **Lifecycle**          | `AppConfig` → bean definitions → creation → injection → destruction. |

---

### 🪞 Core takeaway

> `AnnotationConfigApplicationContext` is Spring’s brain.
> `AppConfig` is its map.
> Together, they let the framework reflectively build, connect, and manage your entire application at runtime.

