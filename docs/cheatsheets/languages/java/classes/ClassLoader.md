---
title: ClassLoader  
date: 2025-10-21  
tags:  
  - java  
  - jvm  
  - classloader  
summary: A complete cheatsheet on Java's ClassLoader mechanism — how the JVM locates, loads, and manages classes via the classpath and loaders.  
aliases:
  - Java ClassLoader
---

# ⚙️ `ClassLoader` — The JVM’s Gatekeeper

> A `ClassLoader` is the bridge between the **bytecode world** (`.class` files, JARs, modules) and the **runtime world** (objects and types).  
> It tells the JVM *where* to find classes and *how* to bring them into memory.

Every class in Java — from `String` to your own — is loaded by exactly one `ClassLoader`.

---

## 1. The Journey: From Source to Runtime

```
source (.java)
↓ compiled by javac
bytecode (.class)
↓ loaded by
ClassLoader
↓ defined in
Class<?> (Metaspace)
↓ instantiated into
Object (Heap)
```

* **ClassLoader** → reads raw bytecode.  
* **Class<?>** → the runtime representation of that type.  
* **Object** → the actual instance living on the heap.

---

## 2. The Loading Chain — Delegation Model

Java’s loaders form a **hierarchy** — each loader delegates to its parent before loading itself.

```
BootstrapClassLoader
↑
PlatformClassLoader (JDK APIs)
↑
AppClassLoader (your code & libs)
↑
Custom Loaders (plugins, frameworks)
```

This model prevents multiple copies of core classes like `java.lang.Object`.  
Only the top-most loader (Bootstrap) defines them once for the entire JVM.

---

## 3. The Classpath — The Loader’s Search Map

> The **classpath** is not a directory — it’s a **runtime search list** that tells the Application ClassLoader where to find `.class` and `.jar` files.

When you run a Java program:

```bash
java -cp "out/:libs/*" com.example.Main
````

or equivalently:

```bash
export CLASSPATH=out/:libs/*
java com.example.Main
```

the JVM passes that list to the **AppClassLoader**, which searches those paths in order.

---

### How the Classpath Works

* Linux/macOS → entries separated by `:`
* Windows → entries separated by `;`
* Each entry can be:

  * a **directory** (searched recursively by package path)
  * a **JAR file**
  * a **wildcard (`*`)** — includes all JARs in a directory

Example:

```
/project/out:/project/lib/*:/opt/javafx/lib/*
```

You can inspect your runtime classpath with:

```java
System.out.println(System.getProperty("java.class.path"));
```

---

### Classpath vs Module Path

| Aspect                | Classpath (Traditional)    | Module Path (Java 9+)           |
| --------------------- | -------------------------- | ------------------------------- |
| Used by               | AppClassLoader             | Module system (JPMS)            |
| Structure             | Flat list of dirs/JARs     | Graph of named modules          |
| Dependency visibility | All-to-all                 | Explicit `requires` / `exports` |
| Common usage          | Spring, legacy apps, tools | Modular apps, JLink runtimes    |

Even modular apps still rely on the **ClassLoader mechanism under the hood** — modules just add structure on top.

---

### Quick Gotchas

* Wrong or missing classpath → `ClassNotFoundException`
* `java -jar` ignores `-cp` unless the JAR’s manifest defines `Class-Path`
* IDEs and build tools (Maven, Gradle, IntelliJ) construct the classpath automatically

---

### Visual Summary

```
+----------------------+
|      Classpath       | ← directories & JARs
+----------------------+
          ↓
     AppClassLoader
          ↓
       JVM Runtime
```

---

## 4. Built-in ClassLoaders

| Loader                   | Loads From                         | Access in Code                         | Example Classes                |
| ------------------------ | ---------------------------------- | -------------------------------------- | ------------------------------ |
| **Bootstrap**            | Core JDK (`rt.jar` / modules)      | `null` (native)                        | `java.lang.*`, `java.util.*`   |
| **Platform (Extension)** | `jre/lib/ext` (old) or JDK modules | `ClassLoader.getPlatformClassLoader()` | `java.sql.*`, `java.xml.*`     |
| **Application (System)** | `CLASSPATH` / `modulepath`         | `ClassLoader.getSystemClassLoader()`   | your app & dependencies        |
| **Custom**               | Anywhere you define                | Subclass of `ClassLoader`              | plugins, agents, hot-reloaders |

Example:

```java
ClassLoader app = ClassLoader.getSystemClassLoader();
System.out.println(app);               // AppClassLoader
System.out.println(app.getParent());   // PlatformClassLoader
System.out.println(app.getParent().getParent()); // null (Bootstrap)
```

---

## 5. How Class Loading Works Internally

Each class passes through these phases:

1. **Loading** — read bytes from `.class`, JAR, or network.
2. **Linking** —

   * *Verification*: bytecode safety checks.
   * *Preparation*: allocate static fields.
   * *Resolution*: replace symbolic refs with direct ones.
3. **Initialization** — run static blocks and field initializers.

You can intercept loading by overriding `findClass(name)`.

---

## 6. Relationship with `Class<?>`

Every loaded class has an associated `Class<?>` object that remembers its loader.

```java
Class<?> c = User.class;
System.out.println(c.getClassLoader()); // AppClassLoader
```

If two identical classes are loaded by different loaders → they are **different types** to the JVM:

```java
boolean same = classA == classB; // false if loaded by different ClassLoaders
```

That’s why app servers (Tomcat, Spring Boot) can isolate multiple apps with identical class names.

---

## 7. Writing a Custom ClassLoader

```java
public class MyLoader extends ClassLoader {
  @Override
  protected Class<?> findClass(String name) throws ClassNotFoundException {
    try {
      byte[] bytes = Files.readAllBytes(Path.of(name.replace('.', '/') + ".class"));
      return defineClass(name, bytes, 0, bytes.length);
    } catch (IOException e) {
      throw new ClassNotFoundException(name, e);
    }
  }
}
```

Usage:

```java
ClassLoader loader = new MyLoader();
Class<?> c = loader.loadClass("com.example.Hello");
Object instance = c.getDeclaredConstructor().newInstance();
```

---

## 8. `loadClass()` vs `findClass()`

| Method            | Role                                         |
| ----------------- | -------------------------------------------- |
| `loadClass(name)` | Handles **delegation** (calls parent first). |
| `findClass(name)` | Defines how *your* loader finds the class.   |

If you override `findClass()`, always call `super.loadClass(name, false)` first to respect delegation.

---

## 9. Why Custom Loaders Exist

| Purpose                   | Example                              |
| ------------------------- | ------------------------------------ |
| **Hot reloading**         | Spring DevTools, JRebel              |
| **Plugin isolation**      | Tomcat webapps, OSGi                 |
| **Dynamic bytecode**      | Hibernate proxies, generated classes |
| **Security / sandboxing** | Applets, custom interpreters         |

---

## 10. Metaspace: Where Classes Live

* Pre-Java 8: **PermGen** held class metadata.
* Since Java 8: **Metaspace** (native memory).
* Stores each loaded class’s structure, methods, constants.
* Classes unload only when:

  * No live references remain, and
  * Their loader is garbage collected.

Long-lived custom loaders can leak memory if they hold on to objects.

---

## 11. Loading From Different Sources

You can load classes from anywhere — filesystem, JARs, or network.

```java
URL[] urls = { new URL("file:/path/to/lib.jar") };
URLClassLoader loader = new URLClassLoader(urls);
Class<?> c = loader.loadClass("com.lib.Tool");
```

Close it when done:

```java
loader.close(); // releases JAR handles
```

---

## 12. Debugging & Introspection

Check which loader loaded a class:

```java
System.out.println(String.class.getClassLoader()); // null (Bootstrap)
System.out.println(User.class.getClassLoader());   // AppClassLoader
```

List all URLs in the system loader:

```java
URLClassLoader cl = (URLClassLoader) ClassLoader.getSystemClassLoader();
for (URL url : cl.getURLs()) System.out.println(url);
```

Print the loader hierarchy:

```java
static void printHierarchy(ClassLoader cl) {
  while (cl != null) {
    System.out.println(cl);
    cl = cl.getParent();
  }
}
```

---

## 13. Class Unloading Rules

A class can be unloaded only when:

1. Its defining `ClassLoader` is unreachable.
2. No live instances of that class remain.
3. No reflective references (`Class<?>`) survive.

In long-lived servers, **leaked loaders = leaked memory**.

---

## 14. Real-World Hierarchies

**Spring Boot Fat JAR:**

```
AppClassLoader
   ↳ LaunchedURLClassLoader (Spring Boot custom)
        ↳ PluginClassLoader (optional)
```

**Tomcat:**

```
CommonLoader
   ↳ CatalinaLoader (server classes)
       ↳ WebAppLoader (per webapp)
```

Each webapp runs in isolation — same class names, different loaders.

---

## 15. ClassLoader + Reflection

Frameworks often pair loaders with reflection:

```java
ClassLoader cl = Thread.currentThread().getContextClassLoader();
Class<?> clazz = cl.loadClass("com.example.ServiceImpl");
Annotation a = clazz.getAnnotation(Service.class);
```

The **Thread Context ClassLoader** lets frameworks load user classes without hardcoding paths.

---

## 16. Common Pitfalls

* Forgetting to close `URLClassLoader` → file lock leaks (especially on Windows).
* Holding references to plugin classes → prevents GC/unloading.
* Violating delegation → `LinkageError`.
* Duplicate classes from different loaders → `ClassCastException: X cannot be cast to X`.
* Misusing context loader → `ServiceLoader` fails to locate providers.

---

## 17. Modern Evolution

* **Java Modules (JPMS)** — adds explicit dependencies; built atop loaders.
* **Instrumentation API** — allows runtime class redefinition.
* **`Lookup#defineClass()` (Java 15+)** — defines classes without subclassing `ClassLoader`.

---

## 18. The Mental Map

```
[ClassLoader]  → defines  →  [Class<?> in Metaspace]
       ↑                         ↓
 hierarchy                    used to create
       ↑                         ↓
 [App/Custom Loaders]     [Objects on Heap]
```

| Concept       | Lives In      | Purpose                       |
| ------------- | ------------- | ----------------------------- |
| `ClassLoader` | Heap          | Loads and defines classes     |
| `Class<?>`    | Metaspace     | Metadata for a type           |
| `Object`      | Heap          | Runtime instance              |
| `Classpath`   | JVM property  | Search map for AppClassLoader |
| `Metaspace`   | Native memory | Class metadata store          |

---

## 19. Quick Reference

| Task              | Code                                    |
| ----------------- | --------------------------------------- |
| System loader     | `ClassLoader.getSystemClassLoader()`    |
| Platform loader   | `ClassLoader.getPlatformClassLoader()`  |
| Loader of a class | `MyClass.class.getClassLoader()`        |
| Get parent loader | `getParent()`                           |
| Define manually   | `defineClass(name, bytes, 0, len)`      |
| Close URL loader  | `close()`                               |
| Get classpath     | `System.getProperty("java.class.path")` |

---

## 20. Final Mind Model

```
                ┌────────────┐
                │ ClassLoader│──────┐ defines
                └─────┬──────┘      ↓
                      │         ┌──────────────┐
                      └────────▶│  Class<?>    │ (Metaspace)
                                └────┬─────────┘
                                     │ instantiates
                                     ↓
                                [Objects on Heap]
```

Everything that exists at runtime passes through this chain.
The classpath feeds it. The ClassLoader enacts it. The JVM sustains it.


## 21. 🔎 Full Method Coverage

Below are the main methods grouped by purpose, including those not yet explicitly mentioned in your version.

---

### 1. Core Loading Methods

| Method                                                        | Description                                                                                                   |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `loadClass(String name)`                                      | Main entry point. Performs parent-delegation before attempting to find the class. Usually **not overridden**. |
| `findClass(String name)`                                      | Child loader’s lookup logic. Usually **overridden** in custom loaders.                                        |
| `defineClass(String name, byte[] b, int off, int len)`        | Converts raw bytecode → `Class<?>`. Also checks security and defines it in the current loader’s namespace.    |
| `defineClass(String name, ByteBuffer b, ProtectionDomain pd)` | Same but accepts a `ByteBuffer`.                                                                              |
| `resolveClass(Class<?> c)`                                    | Links the class (resolves symbolic references). Often called after defining.                                  |
| `findLoadedClass(String name)`                                | Checks whether the class has already been loaded by this loader. Useful to prevent re-definition.             |

---

### 2. Resource Lookup (not classes, but files inside JARs / paths)

| Method                             | Description                                                   |
| ---------------------------------- | ------------------------------------------------------------- |
| `getResource(String name)`         | Finds a single resource (delegates to parent first).          |
| `getResources(String name)`        | Returns all matches (`Enumeration<URL>`).                     |
| `getResourceAsStream(String name)` | Opens resource as stream. Convenient for configs inside JARs. |
| `findResource(String name)`        | Child’s own lookup (skip parent).                             |
| `findResources(String name)`       | Same idea, but for multiple results.                          |
| `getResourceAsStream(String name)` | Handy wrapper combining both.                                 |

**Note:** These work for `.properties`, `.xml`, images, or anything packaged in classpath/JARs — not `.class` files specifically.

---

### 3. Classpath / URL Access (via subclasses)

| Method            | Description                                                                         |
| ----------------- | ----------------------------------------------------------------------------------- |
| `getURLs()`       | (only in `URLClassLoader`) — lists all JARs and directories this loader reads from. |
| `addURL(URL url)` | (protected in `URLClassLoader`) — dynamically extend classpath.                     |
| `close()`         | (since Java 7) — release open JAR handles (critical on Windows).                    |

---

### 4. Context & Hierarchy Methods

| Method                                                       | Description                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------- |
| `getParent()`                                                | Returns parent loader in the delegation chain.                |
| `getName()`                                                  | Human-readable name of this loader (Java 9+).                 |
| `getDefinedPackage(String name)`                             | Returns metadata for a single package defined by this loader. |
| `getDefinedPackages()`                                       | Returns all packages defined by this loader.                  |
| `setDefaultAssertionStatus(boolean enabled)`                 | Controls assertion checking.                                  |
| `setPackageAssertionStatus(String pkg, boolean enabled)`     | Per-package assertions.                                       |
| `setClassAssertionStatus(String className, boolean enabled)` | Per-class assertions.                                         |
| `clearAssertionStatus()`                                     | Resets all assertion settings.                                |

These assertion methods were added when Java introduced `assert` — rarely used today, but still valid.

---

### 5. Security & ProtectionDomain

| Method                                                                      | Description                                                                  |
| --------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `getDefinedPackage(String name)`                                            | Returns info about the package and its signing/sealing metadata.             |
| `definePackage(...)`                                                        | Defines a package with attributes (version, vendor, sealBase).               |
| `getPackage(String name)`                                                   | Legacy method (pre-Java 9). Returns info if the package was already defined. |
| `getPackages()`                                                             | Legacy variant returning all loaded packages.                                |
| `getSystemResource(String name)` / `getSystemResourceAsStream(String name)` | Static utility methods using the **system classloader**.                     |

---

### 6. Thread Context Loader Utilities

| Method                                                 | Description                                                                                            |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `Thread.currentThread().getContextClassLoader()`       | Retrieves the loader used for reflection, frameworks, and service loading.                             |
| `Thread.currentThread().setContextClassLoader(loader)` | Temporarily changes loader for current thread. Used heavily by Spring, Hibernate, and `ServiceLoader`. |

Not part of `ClassLoader` class itself, but conceptually essential.

---

### 7. Internal / Advanced (less common but still relevant)

| Method                            | Description                                                        |
| --------------------------------- | ------------------------------------------------------------------ |
| `defineModule(...)`               | Used by JPMS to define a `Module` for this loader.                 |
| `findModule(...)`                 | Looks up modules known by this loader.                             |
| `getUnnamedModule()`              | Returns the “default” module (non-modular classes).                |
| `registerAsParallelCapable()`     | Declares that loader can safely load in parallel (multi-threaded). |
| `isRegisteredAsParallelCapable()` | Checks the above flag.                                             |

---

### 8. Deprecated / Historical

| Method                                                                                                    | Notes                                                                                        |
| --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `getSystemClassLoader()`                                                                                  | Still valid, but sometimes replaced by `ModuleLayer.boot().findLoader(...)` in modular apps. |
| `getSystemResource*()`                                                                                    | Static variants still used widely.                                                           |
| `defineClass` variants with `ProtectionDomain` and `CodeSource` — older security model, still functional. |                                                                                              |

---

#### Meta-Insight

`ClassLoader` is deceptively deep.
At first glance it looks like just a “read bytes and define classes” utility, but in modern JVMs it also manages:

* Module boundaries (post-Java 9)
* Parallel class loading safety
* Resource abstraction across JARs and layers
* Assertion control and package sealing

The class is *half relic, half skeleton key* — much of the JVM’s modular world still stands on its shoulders.


