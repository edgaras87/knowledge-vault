---
title: Reflection & `@Autowired`
date: 2025-10-20
tags: 
    - java
    - spring
    - reflection
    - autowired
summary: A cheatsheet explaining how Spring uses reflection to discover, instantiate, and wire beans at runtime using the `@Autowired` annotation.
aliases:
  - Spring Reflection & Autowired

---




# 🌱 Spring, Reflection & `@Autowired` — Cheatsheet

### Mental model

Spring uses **reflection** to *discover*, *instantiate*, and *connect* your beans at runtime.
It’s not doing anything “mystical” — just a highly organized, optimized use of Java’s reflection and classloader system.

---

## 🧭 The Big Picture

```
Classpath → ClassLoader → @Component scan → Reflection → Bean creation → @Autowired injection
```

At startup, Spring Boot performs this sequence:

1. **Scans** your classpath for annotated classes.
2. **Loads** those `.class` definitions into memory via the JVM’s ClassLoader.
3. **Creates `Class<?>` objects** in metaspace.
4. **Uses reflection** to:

   * instantiate beans (`newInstance()`)
   * read annotations (`@Component`, `@Autowired`, `@Configuration`, etc.)
   * set fields or call methods to wire dependencies

---

## 🪞 Step-by-step breakdown

### 1️⃣ Classpath scanning

Spring starts with `@SpringBootApplication` → triggers component scanning.

```java
@ComponentScan(basePackages = "com.example")
```

At runtime:

* Spring’s scanner walks through your classpath entries (JARs, folders).
* It finds classes annotated with:

  * `@Component`
  * `@Service`
  * `@Repository`
  * `@Controller`
* It loads those `.class` files and keeps metadata in memory.

No beans are created yet — only **metadata** is registered.

---

### 2️⃣ Bean instantiation (reflection)

Spring now instantiates each discovered bean reflectively.

```java
Class<?> beanClass = PaymentService.class;
Object instance = beanClass.getDeclaredConstructor().newInstance();
```

Then it wraps it in a `BeanDefinition` — a metadata container that stores the class type, scope, and dependencies.

This is done **without you writing any `new` keyword**.
That’s reflection in action.

---

### 3️⃣ Dependency resolution (`@Autowired`)

When Spring encounters a field like:

```java
@Service
public class PaymentService {
    @Autowired
    private AuditService auditService;
}
```

it uses reflection to perform **field injection**:

```java
Field f = PaymentService.class.getDeclaredField("auditService");
f.setAccessible(true);
f.set(paymentServiceInstance, auditServiceInstance);
```

That’s literally what happens behind the scenes.

---

### 4️⃣ Other injection methods

| Type                      | Example                                  | Reflection action                                    |
| ------------------------- | ---------------------------------------- | ---------------------------------------------------- |
| **Field injection**       | `@Autowired private Repo repo;`          | `Field.setAccessible(true); field.set(bean, value);` |
| **Constructor injection** | `@Autowired public Service(Repo repo)`   | Read constructor metadata → invoke with arguments    |
| **Setter injection**      | `@Autowired public void setRepo(Repo r)` | Call method reflectively with dependency             |

All use reflection; only the injection point changes.

---

### 5️⃣ `@Configuration` and `@Bean`

When you define beans in config classes:

```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService();
    }
}
```

Spring creates an instance of `AppConfig` reflectively, then uses reflection again to call the `@Bean` method and register its return value as another bean.

---

## 🧩 Example — Full Reflection Cycle in Spring

```java
@Service
public class PaymentService {
    @Autowired
    private AuditService audit;
}

@Service
public class AuditService {}
```

At runtime:

1. Spring scans → finds `PaymentService` and `AuditService`.
2. For each, it loads the class via ClassLoader.
3. Reflection creates bean instances.
4. Finds `@Autowired` → retrieves the matching dependency.
5. Uses `Field.setAccessible(true)` → injects the dependency.
6. The fully-wired bean enters the ApplicationContext.

After that, all beans behave like normal Java objects — reflection is no longer involved in normal execution.

---

## 🧠 Where `Class<?>` fits in

Spring keeps a `Class<?>` reference for every bean definition:

```java
BeanDefinition bd = ...;
Class<?> type = bd.getBeanClass();
```

This allows it to:

* read annotations dynamically
* call constructors or methods reflectively
* create new instances via the `Class<?>` handle

That’s why `Class<?>` is the “entry ticket” for all reflective operations in the Spring container.

---

## ⚙️ Internals peek — pseudo code

```java
for (Class<?> clazz : scannedClasses) {
    if (clazz.isAnnotationPresent(Component.class)) {
        Object bean = clazz.getDeclaredConstructor().newInstance();
        beans.put(clazz.getSimpleName(), bean);

        for (Field field : clazz.getDeclaredFields()) {
            if (field.isAnnotationPresent(Autowired.class)) {
                Object dependency = findMatchingBean(field.getType());
                field.setAccessible(true);
                field.set(bean, dependency);
            }
        }
    }
}
```

Simplified, but this captures the essence of what Spring’s IoC container does with reflection.

---

## 🧩 ClassLoader in Spring Boot

Spring Boot uses a **custom classloader** called `LaunchedURLClassLoader`
to load nested JARs from inside your fat JAR (`app.jar`).

That’s how it can run with:

```bash
java -jar app.jar
```

without unpacking — the custom loader reads `.class` bytes directly from the JAR.

---

## 🔍 Related annotations and reflection roles

| Annotation       | Purpose                               | Reflection action                  |
| ---------------- | ------------------------------------- | ---------------------------------- |
| `@Component`     | Marks class as Spring-managed bean    | Discovered via reflection scan     |
| `@Autowired`     | Marks dependency to inject            | Field or method reflection         |
| `@Configuration` | Marks class providing `@Bean` methods | Reflection to call config methods  |
| `@Bean`          | Defines bean factory method           | Reflection invocation              |
| `@Value`         | Injects property value                | Reads field or setter reflectively |
| `@PostConstruct` | Runs method after injection           | Reflective method call             |

---

## 🧩 Common pitfalls

| Problem                            | Root cause                                   | Fix                                                                                    |
| ---------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------- |
| `NoSuchBeanDefinitionException`    | No matching bean type found for `@Autowired` | Check package scanning / qualifiers                                                    |
| `BeanCurrentlyInCreationException` | Circular dependency between beans            | Refactor to constructor or setter injection                                            |
| `IllegalAccessException`           | Private field without `setAccessible(true)`  | Spring handles this internally — but shows up in proxies if security manager blocks it |
| `ClassNotFoundException`           | Bean class missing from classpath            | Add dependency or correct package name                                                 |

---

## ⚡ Performance tricks

* Spring caches **every reflective lookup** (constructor, field, annotation).
* It never scans the classpath on every request — only once at startup.
* Once the ApplicationContext is built, runtime overhead is negligible.

You can confirm this in stack traces — reflection appears only during context bootstrap.

---

## 🧩 Reflection in proxies (bonus)

When you use `@Transactional`, `@Async`, or `@Cacheable`, Spring builds **proxy objects** around your beans using reflection and bytecode generation (via CGLIB or JDK proxies).

Those proxies override methods dynamically to insert logic like transactions or logging — again, pure runtime reflection & bytecode magic.

---

## 🧭 TL;DR Summary

| Concept                | Description                                   |
| ---------------------- | --------------------------------------------- |
| **Classpath scanning** | Finds annotated classes                       |
| **ClassLoader**        | Loads `.class` bytes into JVM                 |
| **Reflection**         | Creates and wires bean objects                |
| **`@Autowired`**       | Dependency injection using reflection         |
| **ApplicationContext** | Registry of fully-wired beans                 |
| **Performance**        | Reflection used only at startup; cached after |

---

### 🧩 One-sentence anchor

> Spring doesn’t break Java’s rules — it just uses reflection masterfully to **discover**, **create**, and **connect** your objects at runtime.

