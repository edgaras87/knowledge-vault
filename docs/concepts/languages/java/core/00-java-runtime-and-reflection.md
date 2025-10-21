---
title: Java Runtime & Reflection
date: 2025-10-20
tags: 
    - java
    - jvm
    - reflection
    - class
    - spring
summary: A quick starter guide to understanding Java's runtime environment, JVM memory structure, reflection mechanism, and how frameworks like Spring leverage these concepts.
aliases:
    - Java Runtime & Reflection

---




# ⚙️ Java Runtime & Reflection — Quick Starter

A guided chain from **JVM internals → runtime behavior → reflection → Spring magic**.

---

## 1. JVM — The Beating Heart of Java

The **Java Virtual Machine (JVM)** is the engine that runs your compiled Java code.
It doesn’t run `.java` source files directly. You write code, the **compiler (`javac`)** translates it into **bytecode (`.class` files)**, and the JVM interprets or JIT-compiles that bytecode into real CPU instructions.

Think of the JVM as:

* A **simulated computer inside your computer**.
* It handles memory, threads, exceptions, and cross-platform behavior.

The JVM is *platform-dependent*, but Java bytecode is *platform-independent* — that’s how Java stays “write once, run anywhere”.

---

## 2. JVM Memory — Heap and Metaspace

Inside the JVM, memory is divided into several regions. The most important:

### 🧠 Heap

Where **objects live**.

* Created when you use `new`.
* Garbage Collected (GC) when no references remain.
* Divided into generations: **Young (Eden + Survivor)** and **Old (Tenured)** space.

### 📘 Metaspace

Where **class metadata** lives (replaced “PermGen” from Java 8+).

* Stores the structure of each loaded class (its methods, fields, annotations, etc.).
* Grows dynamically; not part of the heap.

### 🧩 Others (for completeness)

* **Stack** — each thread gets its own, holds method calls and local variables.
* **PC Register** — keeps track of which instruction a thread is executing.
* **Native Method Area** — used when calling C/C++ code through JNI.

---

## 3. Object — The Root of Everything

Everything that lives on the heap is an **Object** (or derived from it).

`Object` is the **universal superclass** — the ancestor of all Java classes.
It defines the methods that all Java entities inherit: `toString()`, `equals()`, `hashCode()`, etc.

When you create an object:

```java
User user = new User("Alice");
```

You’re creating an **instance** in the heap, and the JVM knows *which class definition* it belongs to — that definition lives in **Metaspace**.

---

## 4. Class<?> — The Blueprint of a Type

Every loaded class in Java has a corresponding **`Class<?>` object** in memory.

This object lives in the Metaspace and acts like the **blueprint** for all instances of that type.
You can access it like this:

```java
Class<?> clazz = User.class;
```

This `clazz` object contains:

* The class’s name
* Methods
* Fields
* Constructors
* Annotations
* And the **ClassLoader** that loaded it

This is the foundation of **reflection** — it lets you **inspect and modify code at runtime**.

---

## 5. ClassLoader — The Bridge Between Disk and Memory

The **ClassLoader** is what **loads class bytecode** into memory and turns it into `Class<?>` objects.

Hierarchy of loaders:

* **Bootstrap** — loads core JDK classes (like `java.lang.*`)
* **Platform** — loads extension libraries
* **Application** — loads your app’s classes from the classpath
* (Sometimes frameworks create custom loaders — like in Spring Boot or Tomcat)

Think of ClassLoader as:

> The librarian that finds your `.class` file, reads it, and registers its “blueprint” in Metaspace.

You can even get it:

```java
ClassLoader loader = User.class.getClassLoader();
```

---

## 6. Reflection — Java Looking at Itself

**Reflection** lets your code inspect and manipulate itself at runtime.

Using the `Class<?>` object, you can:

* List all methods, fields, and constructors.
* Get or set field values dynamically.
* Call methods by name (even private ones, if you bypass access checks).

Example:

```java
Class<?> clazz = User.class;
Method m = clazz.getMethod("getName");
Object result = m.invoke(new User("Alice"));
```

That line means: *“Find method getName in class User and call it.”*

Reflection is powerful but also **slower** and **riskier**, because it bypasses compile-time checks.

---

## 7. JDK, JRE, and JVM — The Holy Trinity

To run Java programs, you need these three players:

| Component                          | What it is             | Contains                |
| ---------------------------------- | ---------------------- | ----------------------- |
| **JVM**                            | The virtual machine    | Executes bytecode       |
| **JRE (Java Runtime Environment)** | JVM + core libraries   | Lets you run programs   |
| **JDK (Java Development Kit)**     | JRE + compiler + tools | Lets you build programs |

So:

* If you only **run** apps → you need JRE.
* If you **develop** apps → you need JDK.
* JVM is what’s *actually executing* inside both.

---

## 8. Spring — Reflection as a Superpower

Spring is built *on top of* the reflection and classloading mechanisms.

### How it uses reflection:

1. **Dependency Injection (DI)** — Finds constructors, fields, and methods annotated with `@Autowired`, and injects dependencies dynamically.
2. **Annotation Scanning** — Uses reflection to detect annotations like `@Component`, `@Service`, etc.
3. **Proxy Creation (AOP)** — Dynamically wraps beans with proxy classes at runtime.
4. **Configuration** — Reads `@Configuration` and `@Bean` annotations to register beans in the ApplicationContext.

When Spring starts, it:

* Scans your classpath
* Uses **ClassLoaders** to find `.class` files
* Builds a map of all annotated classes (via reflection)
* Creates bean instances and wires them together

So Spring is basically a **meta-program that analyzes and constructs your program** using the very same JVM tools you’ve now met.

---

## 9. Putting It All Together — The Flow

```
source code (.java)
       ↓  javac
bytecode (.class)
       ↓  ClassLoader
Class<?> in Metaspace
       ↓  Reflection
Objects on Heap
       ↓  Framework Magic
Spring uses Reflection → Builds App Context → Injects Dependencies
```

Everything connects:

* **JVM** runs the show
* **Heap** stores living instances
* **Metaspace** stores their blueprints
* **ClassLoader** loads those blueprints
* **Reflection** lets code see and shape those blueprints
* **Spring** automates that reflection to build entire systems dynamically

---

### Final Thought

Java was designed for *portability* and *safety*, yet its runtime is flexible enough to rewrite its own behavior on the fly.
That balance — rigid typing at compile-time, elastic introspection at runtime — is what allows frameworks like Spring to exist at all.


---

```aiignore





```

---

# 🧭 Java Runtime & Reflection — Comprehensive Quick Starter

A journey from the inner gears of the **JVM** to the elegant machinery of **Spring**, connecting how memory, class loading, and reflection form the foundation of modern Java.

---

## 1. JVM — The Living Machine Beneath Java

When you run a Java program, you’re not running it *directly* on your CPU.
Your code first gets compiled into **bytecode**, an intermediate language designed not for any particular processor, but for a **virtual one**: the Java Virtual Machine, or **JVM**.

The JVM is the **living organism** at the center of all Java execution. It is the software abstraction of a computer — complete with memory management, execution engine, garbage collector, and a few tricks that physical CPUs can’t even dream of.

It reads `.class` files, interprets or JIT-compiles them into native instructions, and handles all the lower-level chaos: memory allocation, stack frames, threading, and exception handling.

Its most remarkable property is **portability**. You can compile your Java program once and run it anywhere a JVM exists — on Windows, Linux, macOS, Android, or even embedded devices.
The physical hardware disappears; what remains is a consistent, predictable virtual world.

---

## 2. JVM Memory — The World Inside the Machine

Like any living system, the JVM has **organs** — specialized memory areas that keep everything running.

### The Heap — The Space of Living Objects

The **heap** is where every object you create with `new` lives.
It’s dynamic, expanding and contracting as objects are born and die.
The **garbage collector** roams this space, reclaiming memory from objects that no longer have references.

To keep things efficient, the heap is divided into regions:

* **Young Generation**, where newborn objects are allocated.
* **Old Generation**, where survivors of many GC cycles live longer lives.

### Metaspace — Where Blueprints Reside

If the heap holds the living objects, **Metaspace** holds their **blueprints**.
When the JVM loads a class, its structure — fields, methods, annotations — is stored in Metaspace.
Unlike the old PermGen space, Metaspace grows dynamically, constrained only by system memory.

Every class in Java has a single shared representation in Metaspace.
All instances (the objects on the heap) refer back to that single blueprint.

### Stacks and Other Regions

Each thread has its own **stack**, which holds local variables and method call frames.
There’s also the **Program Counter (PC) register**, keeping track of which bytecode instruction a thread is executing.
And then there’s the **native method area**, where code written in C/C++ and called via JNI resides.

All these parts work together like organs in a body — the heap breathing objects in and out, the stack pulsing with method calls, and Metaspace serving as the collective memory of what "things" exist.

---

## 3. Object — The First Citizen of the Heap

In Java, everything that exists in the heap — every `String`, every `ArrayList`, every `User` — ultimately descends from one ancestor: `java.lang.Object`.

That single class is the root of the entire Java type system.
It defines the fundamental behaviors shared by all entities — the ability to be compared (`equals`), described (`toString`), and hashed (`hashCode`).

When you write:

```java
User user = new User("Alice");
```

you’re creating an **instance**, an individual thing in the heap that conforms to the **blueprint** of the `User` class stored in Metaspace.
The JVM knows exactly which blueprint this object follows and which methods it can use, because of the internal pointer from the object’s header to its class metadata.

That connection between *object instance* and *class blueprint* is the bridge between the runtime world (heap) and the static world (code).

---

## 4. Class<?> — The Blueprint of Existence

When Java loads a class, it creates a special object in Metaspace: an instance of `java.lang.Class`.
Yes, the definition of a class is itself represented by an object.
This is one of Java’s most elegant ideas — **everything, even types, is data**.

You can get hold of it like this:

```java
Class<?> clazz = User.class;
```

That `clazz` object carries every detail about the class: its name, modifiers, methods, fields, annotations, constructors, and which loader brought it into existence.

This is why the type parameter looks odd — `Class<?>`. The question mark (`?`) means “I don’t know what specific type this represents, but it’s a class of *something*.”

Each loaded class has exactly one `Class` object representing it.
All instances of that class refer back to the same `Class` object.
That’s how the JVM maintains order and consistency: a single canonical definition per loaded type.

---

## 5. ClassLoader — The Librarian of the JVM

Where do those `Class<?>` blueprints come from?
From the **ClassLoader**, Java’s librarian.

The ClassLoader is responsible for finding `.class` files, reading their bytecode, and turning them into live `Class` objects stored in Metaspace.

There isn’t just one — ClassLoaders form a **hierarchy**:

* **Bootstrap Loader** loads core Java classes from the JDK (`java.lang`, `java.util`, etc.).
* **Platform Loader** loads JDK extensions and libraries.
* **Application Loader** loads your project’s classes from the classpath.

Frameworks often create **custom classloaders** — for example, Tomcat and Spring Boot both do this to isolate applications or support hot reloading.

When a class is needed, the JVM asks the appropriate ClassLoader: *“Do you have this one?”*
If not, it delegates to its parent. Only when the parent can’t provide it does it load the class itself.
This mechanism guarantees that system classes remain consistent while still allowing user code to define its own world.

---

## 6. Reflection — Java Examining Itself

Reflection is Java’s self-awareness.
It’s the ability of a running program to **inspect and manipulate its own structure** — to look at classes, methods, and fields, and even invoke them dynamically.

This capability comes from those `Class<?>` objects sitting in Metaspace.
Once you have a `Class<?>`, you can start exploring:

```java
Class<?> clazz = User.class;
Method m = clazz.getDeclaredMethod("getName");
Object result = m.invoke(new User("Alice"));
```

This is Java code calling methods by **name**, discovered at runtime, not compile time.
It’s slower and more dangerous — because you lose static type safety — but extraordinarily powerful.

Reflection enables frameworks to perform **meta-programming**: code that writes or modifies other code dynamically.

---

## 7. JDK, JRE, and JVM — The Trinity of Execution

At this point it’s worth separating the pieces you install and run:

* The **JVM** is the core engine that executes bytecode.
* The **JRE** (Java Runtime Environment) packages the JVM with standard libraries and runtime support — enough to *run* Java applications.
* The **JDK** (Java Development Kit) includes everything in the JRE plus compilers (`javac`), debugging tools, and build utilities — enough to *develop* applications.

So:

* Users of Java programs need the **JRE**.
* Developers of Java programs need the **JDK**.
* Both rely on the **JVM**, which is where the bytecode lives and breathes.

They form a nested hierarchy:

```
JDK = JRE + compiler + tools
JRE = JVM + standard libraries
```

---

## 8. Spring — Reflection Turned Into Engineering

Everything that Spring does — dependency injection, configuration, AOP — rests on reflection and classloading.

When you start a Spring application, it performs a grand act of **introspection**:

1. It scans the **classpath**, asking ClassLoaders to enumerate every `.class` file available.
2. It uses **reflection** to inspect those classes for specific annotations: `@Component`, `@Service`, `@Controller`, `@Configuration`.
3. It constructs an internal registry (the **ApplicationContext**) that maps out all known classes and their dependencies.
4. It instantiates those classes dynamically using reflection, often injecting one into another using constructor or field injection.
5. It sometimes wraps them in **proxies** — dynamic subclasses or interfaces that intercept method calls to provide extra behavior (like transaction management or logging).

Every piece of this process — scanning, loading, wiring — depends on the JVM’s built-in meta-system.
Spring is not magic; it’s a disciplined orchestration of classloaders, metadata, and reflection.
It turns static code into a **living, configurable organism**.

---

## 9. The Whole Flow — From Code to Context

Here’s the conceptual journey in one unbroken line:

1. You write `.java` source files.
2. `javac` compiles them into `.class` bytecode.
3. ClassLoaders pull that bytecode into the JVM and store class metadata in Metaspace.
4. The JVM uses those blueprints to create objects on the heap.
5. Reflection allows runtime access and modification of those blueprints.
6. Spring leverages reflection to wire up and manage those objects automatically.

The entire stack — from low-level memory regions to high-level frameworks — is one continuous spectrum.
The heap is the living matter.
Metaspace is the genetic code.
ClassLoaders are the reproduction system.
Reflection is the consciousness.
Spring is the civilization that emerges on top.

---

## 10. Why It Matters

Understanding this chain transforms how you read errors, optimize performance, and design systems.
When you see a `ClassNotFoundException`, you know it’s a **ClassLoader problem**.
When you hit `OutOfMemoryError: Metaspace`, you know it’s a **class metadata leak**, not a heap issue.
When you step into a Spring bean and see it’s a proxy class, you can trace the reflection trail back to the original.

At its best, Java is not just a language — it’s a small self-contained universe.
Its runtime introspection gives it a quality rarely found in compiled languages: the ability to evolve itself at runtime.
That’s what makes Spring, Hibernate, and every major framework possible.

