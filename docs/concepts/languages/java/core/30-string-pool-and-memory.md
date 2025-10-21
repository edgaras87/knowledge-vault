---
title: String Pool & JVM Memory Model
date: 2025-10-21
tags: 
    - java
    - jvm
    - string
    - memory-model
summary: An explanation of how Java's String pool fits into the JVM memory model, including its relationship to objects and class metadata.
aliases:
    - String Pool & JVM Memory Model

---



# 🧩 How Strings Fit into the JVM’s Memory Model

---

## 1️⃣ Big Picture — The Three Key Memory Zones

When the JVM runs, it divides its runtime memory roughly like this:

```
+-----------------------------------------------+
| JVM Process Memory                            |
|                                               |
|  ┌──────────────┐  ┌──────────────┐           |
|  |  Heap        |  |  Metaspace   |           |
|  |  (objects)   |  |  (class info)|           |
|  └──────────────┘  └──────────────┘           |
|          ▲                 ▲                  |
|          │                 │                  |
|          │                 │                  |
|     Object instances   Class<?> blueprints    |
|                                               |
|  +-----------------------------------------+  |
|  |  Stack (per thread)                     |  |
|  |  → holds local vars, call frames, refs  |  |
|  +-----------------------------------------+  |
+-----------------------------------------------+
```

* **Heap:** where live *objects* (like `new User()` or `new String()`) live.
* **Metaspace:** where *class metadata* (`Class<?>`) is stored.
* **Stack:** where *temporary references* and *method calls* are tracked per thread.

So far so good.

Now — where does the “String pool” live in all this?

---

## 2️⃣ The String Pool — a Special Corner of the Heap

When you write:

```java
String s1 = "Hello";
String s2 = "Hello";
```

you don’t get two `String` objects.
Both `s1` and `s2` point to the same one.

That’s because of the **String pool** — a cache of unique string literals maintained by the JVM inside the **heap** (specifically in a small internal structure sometimes called the *interned string table*).

---

### 🧠 Why it exists

Strings are everywhere in code — class names, annotations, log messages, JSON keys.
If every identical literal created a new object, memory would explode.

The pool ensures that **identical string literals share one object**.

---

### 🔧 How it works

* When the JVM loads a class from bytecode, it parses constant values (including string literals).
* For every `"Hello"` literal, it checks a hidden table in the heap.

  * If `"Hello"` exists already → reuse the same object.
  * If not → create it once and store it in that table.
* That table is called the **string intern pool**.

---

### 🧩 Visual idea

```
Heap
 ├── String Pool
 │     ├── "Hello"   ← shared
 │     └── "World"   ← shared
 ├── Other Objects
 │     ├── new User()
 │     └── new ArrayList()
```

Both

```java
String a = "Hello";
String b = "Hello";
```

point to the same `"Hello"` inside the pool.

---

## 3️⃣ How it connects to Objects and Classes

| Concept             | Description                         | Where it lives          |
| ------------------- | ----------------------------------- | ----------------------- |
| `String` object     | an instance of `java.lang.String`   | Heap                    |
| `"literal"`         | loaded and pooled at class loading  | Heap (string pool)      |
| `String.class`      | the blueprint of all String objects | Metaspace               |
| `ClassLoader`       | loads `java.lang.String` at startup | Bootstrap classloader   |
| `String s = "abc";` | references a pooled String          | Heap (shared entry)     |
| `new String("abc")` | explicitly makes a new object       | Heap (outside the pool) |

So, `String` behaves like any other `Object`,
but it’s *interned* — meaning the JVM manages a special table of “known identical values.”

---

## 4️⃣ The role of `intern()`

You can manually play with the pool:

```java
String a = new String("Hello");
String b = a.intern();

System.out.println(a == b); // false (a is new)
System.out.println(b == "Hello"); // true (pooled)
```

* `new String("Hello")` → creates a brand-new object on the heap, outside the pool.
* `a.intern()` → checks the pool:

  * if `"Hello"` exists → return reference to that pooled one,
  * otherwise add it and return it.

So `intern()` bridges your runtime-created strings back into the shared pool.

---

## 5️⃣ Historical context — PermGen vs Metaspace

Before Java 8, the **string pool** lived in *PermGen* (an older memory area used for class metadata).
This caused frequent `OutOfMemoryError: PermGen space` issues when too many strings were interned.

In Java 8+, the pool was moved to the **heap**,
and *PermGen* was replaced by **Metaspace**, which stores class metadata (`Class<?>` objects).

That’s why today:

```
Strings → Heap
Class<?> → Metaspace
```

Different worlds, but both part of the JVM runtime memory.

---

## 6️⃣ Relation to Class Loaders

When a class is loaded, all its *string literals* are loaded along with it.
So the ClassLoader that loads your class also triggers the JVM to register its string constants into the pool.

Example:

```java
public class Demo {
    private static final String GREETING = "Hello";
}
```

* When `Demo.class` is loaded:

  * The `ClassLoader` parses bytecode constants.
  * Finds `"Hello"` → checks the string pool.
  * Interns it (adds to the shared table if not already there).

That’s why string literals get pooled automatically *at class load time.*

---

## 7️⃣ Putting it all together — unified model

```
                    JVM MEMORY MAP
+------------------------------------------------------------+
|                        JVM Memory                          |
|------------------------------------------------------------|
|                         Heap                               |
|   ┌──────────────────────────────────────────────────────┐  |
|   │ Object Instances (new User(), new ArrayList(), ...)  │  |
|   │ String Pool: "Hello", "World", "abc", ... (shared)   │  |
|   └──────────────────────────────────────────────────────┘  |
|                                                            |
|                         Metaspace                          |
|   ┌──────────────────────────────────────────────────────┐  |
|   │ Class<?> objects for: User, String, ArrayList, etc.  │  |
|   │ Each describes structure & behavior of its type.     │  |
|   └──────────────────────────────────────────────────────┘  |
|                                                            |
|                         Stack (per thread)                 |
|   ┌──────────────────────────────────────────────────────┐  |
|   │ Method frames, locals, references to heap objects.   │  |
|   └──────────────────────────────────────────────────────┘  |
+------------------------------------------------------------+
```

**Arrows of connection:**

* Every object on the **heap** knows its `Class<?>` in **metaspace**.
* Every string literal is shared through the **string pool** (inside the heap).
* Class loaders read class files and register both **Class<?> metadata** and **string literals**.

---

## 🧩 TL;DR Summary

| Concept          | Description                              | Memory Area           |
| ---------------- | ---------------------------------------- | --------------------- |
| `Object`         | Live instance with fields                | Heap                  |
| `Class<?>`       | Metadata blueprint                       | Metaspace             |
| `ClassLoader`    | Reads `.class` bytes, defines `Class<?>` | Code area → Metaspace |
| `String literal` | Interned, shared value                   | Heap (string pool)    |
| `new String()`   | Non-pooled string                        | Heap (normal object)  |
| `intern()`       | Returns pooled version of a string       | Heap (shared table)   |

---

### 🪞 One-sentence anchor

> The **String pool** is not a magical dimension — it’s just a shared cache of `String` objects in the heap, built automatically when classes are loaded, while `Class<?>` lives separately in metaspace describing *how* strings and all other objects exist.

