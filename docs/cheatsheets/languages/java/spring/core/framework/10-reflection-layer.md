---
title: Reflection Layer  
date: 2025-10-21
tags: 
    - java
    - spring
    - reflection
    - autowired
summary: A cheatsheet explaining how Spring uses reflection to discover, instantiate, and wire beans at runtime.
aliases:
  - Spring Reflection Layer
---

# 🪞 Reflection Layer — How Spring Sees and Wires Your Code

> **Essence:**
> The *Reflection Layer* is the foundation of Spring.
> It’s how Spring **discovers**, **creates**, and **connects** your classes at runtime — turning plain Java code into a self-managing system.

---

## 🌱 1. Concept Overview

Java’s **Reflection API** lets code inspect and manipulate itself at runtime.
Spring builds on that to power `@ComponentScan`, `@Autowired`, `@Bean`, and most of its container magic.

```
source (.java)
   ↓ compiler
bytecode (.class)
   ↓ ClassLoader
Class<?> object in Metaspace
   ↓ Reflection
Spring inspects, instantiates, and injects beans
```

---

## 🧩 2. Core Reflection API — Java’s Runtime Mirror

Everything starts with the `Class<?>` object.

```java
Class<?> clazz = PaymentService.class;
Class<?> clazz2 = Class.forName("com.app.PaymentService");
```

### 🔍 Inspecting a Class

```java
clazz.getName();           // "com.app.PaymentService"
clazz.getDeclaredFields(); // all fields
clazz.getDeclaredMethods();
clazz.getSuperclass();
clazz.getAnnotations();
```

Spring uses this to find annotations like `@Component`, `@Service`, or `@Bean`.

---

## 🧠 3. Creating Objects Reflectively

```java
Constructor<?> ctor = clazz.getDeclaredConstructor();
Object instance = ctor.newInstance();
```

With parameters:

```java
Constructor<?> ctor = clazz.getDeclaredConstructor(AuditService.class);
Object obj = ctor.newInstance(new AuditService());
```

That’s what Spring does when it creates beans from your classes —
it calls constructors reflectively based on metadata from scanning.

---

## ⚙️ 4. Accessing Private Members

Reflection can bypass visibility restrictions.

```java
Field f = clazz.getDeclaredField("audit");
f.setAccessible(true);
f.set(instance, new AuditService());
```

This is the essence of **dependency injection** — Spring sets fields without needing public setters.

---

## 🧬 5. Invoking Methods

```java
Method m = clazz.getDeclaredMethod("process", String.class);
m.setAccessible(true);
m.invoke(instance, "Alice");
```

Used internally for:

* Lifecycle hooks (`@PostConstruct`, `@PreDestroy`)
* Configuration methods (`@Bean`)
* Event listeners (`@EventListener`)

---

## ⚡ 6. Annotations in Reflection

```java
if (clazz.isAnnotationPresent(Service.class)) {
    Annotation a = clazz.getAnnotation(Service.class);
}
```

Field or method level:

```java
Field f = clazz.getDeclaredField("repo");
if (f.isAnnotationPresent(Autowired.class)) {
    // inject dependency reflectively
}
```

Reflection lets Spring read annotation metadata and decide what to do with it.

---

## 🤝 7. `@Autowired` — Reflection Meets IoC

`@Autowired` is the **declarative interface** to reflection.
It tells Spring: *“Find me something of this type and assign it here.”*

### Field Injection

```java
@Autowired
private AuditService audit;
```

Spring finds a matching bean and executes:

```java
Field f = PaymentService.class.getDeclaredField("audit");
f.setAccessible(true);
f.set(paymentServiceInstance, auditInstance);
```

---

### Constructor Injection

```java
@Autowired
public PaymentService(AuditService audit) {
    this.audit = audit;
}
```

If there’s only one constructor, `@Autowired` is optional — Spring assumes it.

Under the hood:

```java
Constructor<?> ctor = clazz.getDeclaredConstructor(AuditService.class);
Object bean = ctor.newInstance(auditBean);
```

---

### Setter Injection

```java
@Autowired
public void setAuditService(AuditService audit) {
    this.audit = audit;
}
```

Spring reflectively calls:

```java
Method m = clazz.getDeclaredMethod("setAuditService", AuditService.class);
m.invoke(instance, auditBean);
```

---

## 🧮 8. Dependency Resolution Rules

When resolving an `@Autowired` dependency:

1. **By Type** — find one matching bean in the ApplicationContext.
2. **If multiple found:**

   * Prefer `@Primary`
   * Or match name with field/parameter
   * Or use `@Qualifier("beanName")`
3. **If none found:**

   * Throw `NoSuchBeanDefinitionException`
   * Unless `@Autowired(required = false)` is used

---

### Qualifiers and Primary

```java
@Primary
@Service
public class EmailNotifier implements Notifier {}

@Service("smsNotifier")
public class SmsNotifier implements Notifier {}

@Service
public class PaymentService {
    @Autowired
    @Qualifier("smsNotifier")
    private Notifier notifier;
}
```

---

### Optional & Lazy Variants

```java
@Autowired(required = false)
private Optional<AuditService> audit;

@Autowired
@Lazy
private HeavyService heavy; // proxy until first call
```

---

## 🧭 9. Internal Flow (Simplified Pseudocode)

```java
for (Field field : clazz.getDeclaredFields()) {
    if (field.isAnnotationPresent(Autowired.class)) {
        Object dependency = context.getBean(field.getType());
        field.setAccessible(true);
        field.set(beanInstance, dependency);
    }
}
```

Spring does this reflection once during startup — then caches results for efficiency.

---

## 🧱 10. Reflection in Bean Lifecycle

```
Scan classes → detect annotations → create Class<?> objects
→ register BeanDefinitions → construct beans via reflection
→ inject dependencies via reflection
→ invoke @PostConstruct → ready
```

Reflection is used heavily during startup —
once the context is ready, execution runs as normal Java code.

---

## ⚙️ 11. Dynamic Proxies (Advanced Reflection)

Spring uses reflection again to generate **proxies** for beans with AOP features:

```java
MyService proxy = (MyService) Proxy.newProxyInstance(
    clazz.getClassLoader(),
    new Class[]{MyService.class},
    (proxyObj, method, args) -> {
        System.out.println("Before call");
        return method.invoke(targetBean, args);
    });
```

Used for:

* Transactions (`@Transactional`)
* Async execution (`@Async`)
* Caching (`@Cacheable`)
* Security (`@PreAuthorize`)

---

## 🧩 12. Common Pitfalls

| Symptom                           | Likely Cause                 | Fix                            |
| --------------------------------- | ---------------------------- | ------------------------------ |
| `ClassNotFoundException`          | Wrong class path             | Verify package name            |
| `NoSuchFieldException`            | Typo or renamed field        | Check reflection target        |
| `NoUniqueBeanDefinitionException` | Multiple beans of same type  | Use `@Qualifier` or `@Primary` |
| Field remains `null`              | Not in `@ComponentScan` path | Fix scan base package          |
| `@Autowired` on static field      | Not supported                | Avoid static injection         |

---

## 🧭 13. Performance Considerations

* Reflection is **slower** than direct calls, but Spring caches all reflective lookups.
* The reflection cost happens only once — at startup.
* During runtime, calls go through direct references (or proxies).

---

## 🧩 14. Quick Reference Summary

| Task                    | API / Annotation                  | Used In Spring For        |
| ----------------------- | --------------------------------- | ------------------------- |
| Discover class metadata | `Class<?>`                        | Component scanning        |
| Create instance         | `Constructor.newInstance()`       | Bean instantiation        |
| Inject dependency       | `Field.set()` / `Method.invoke()` | `@Autowired` wiring       |
| Access annotations      | `isAnnotationPresent()`           | Detect Spring stereotypes |
| Build proxies           | `Proxy.newProxyInstance()`        | AOP wrapping              |
| Resolve dependency      | `@Qualifier`, `@Primary`          | Ambiguity handling        |
| Delay creation          | `@Lazy`                           | Lazy bean proxies         |

---

### 🔗 Related

* [Concept: Reflection & @Autowired](../../../../concepts/frameworks/spring/core/05-reflection-and-autowired.md)
* [Cheatsheet: bean-lifecycle.md](../beans/bean-lifecycle.md)
* [Cheatsheet: aop-proxy-types.md](../aop/aop-proxy-types.md)

---

### 🪞 Core Takeaway

> **Reflection gives Spring its eyes; `@Autowired` gives it its hands.**
> Together they let the framework *see* your classes, *build* your objects,
> and *connect* them into a living application — all before your code ever runs.

