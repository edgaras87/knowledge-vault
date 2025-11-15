---
title: record
date: 2025-11-15
tags: 
    - java
    - language
    - core
summary: Comprehensive practical cheatsheet for Java's `record` feature, covering syntax, usage scenarios, constructors, methods, immutability, pattern matching, and best practices.
aliases:
  - Java — record Cheatsheet

---

# Java `record` — Practical Cheatsheet

## Content

- [0. What is a `record`?](#0-what-is-a-record)
- [1. Basic syntax & rules](#1-basic-syntax--rules)
- [2. When to use a record](#2-when-to-use-a-record)
- [3. Constructors: canonical vs compact](#3-constructors-canonical-vs-compact)
- [4. Methods and interfaces](#4-methods-and-interfaces)
- [5. Immutability and “withers”](#5-immutability-and-withers)
- [6. equals, hashCode, toString](#6-equals-hashcode-tostring)
- [7. Inheritance & sealed types](#7-inheritance--sealed-types)
- [8. Pattern matching with records](#8-pattern-matching-with-records)
- [9. Records vs normal classes](#9-records-vs-normal-classes)
- [10. Records with generics](#10-records-with-generics)
- [11. Local records](#11-local-records)
- [12. Records & frameworks (Jackson, Spring, JPA)](#12-records--frameworks-jackson-spring-jpa)
- [13. Serialization & performance notes](#13-serialization--performance-notes)
- [14. Mental checklist](#14-mental-checklist)

---

## 0. What is a `record`?

A **record** is a special kind of class whose main purpose is to hold data with:

- `private final` fields
- a canonical constructor
- accessors (getters without `get`)
- `equals(..)`, `hashCode()`, `toString()`

```java
public record Point(int x, int y) { }

// usage
Point p = new Point(10, 20);
int xx = p.x();         // accessor (no getX())
System.out.println(p);  // Point[x=10, y=20]
```

Think: tiny immutable data carrier with built-in value semantics.

---

## 1. Basic syntax & rules

### 1.1 Declaration

```java
public record User(String username, String email, int age) { }
```

Roughly equivalent to a final class with:

* final fields
* constructor
* accessors
* equals/hashCode/toString

### 1.2 Restrictions / properties

* Implicitly `final` (you **cannot** extend a record).
* Implicitly extends `java.lang.Record`.
* May **implement interfaces**, but not extend a class.
* Components become `private final` fields; **no setters**.
* You cannot add extra instance fields (only `static`).

```java
public record Config(String env) {

    public static final String DEFAULT_ENV = "dev"; // OK

    // private String mutable;  // ❌ not allowed, extra instance field
}
```

---

## 2. When to use a record

Use a record when the type is:

* **Mostly data** (value object, DTO, config, response shape).
* Conceptually **immutable**.
* Identity is “all fields” (value-based equality).

Examples:

```java
public record Money(String currency, long cents) { }
public record Coordinates(double lat, double lon) { }
public record UserSummary(long id, String name, String email) { }
```

Avoid records for:

* JPA entities (need mutability, no-arg constructors, proxies).
* Objects with identity not based on all fields (e.g. entity with DB id, but equality only by id).

---

## 3. Constructors: canonical vs compact

### 3.1 Canonical constructor

**Canonical constructor** has the same parameters as the record header:

```java
public record User(String username, String email, int age) {

    public User(String username, String email, int age) {
        if (age < 0) {
            throw new IllegalArgumentException("age must be >= 0");
        }
        this.username = username;
        this.email = email;
        this.age = age;
    }
}
```

Once you define it, you must assign all fields manually.

### 3.2 Compact constructor (validation-friendly)

Compact form omits the parameter list; the compiler injects parameters and assignments:

```java
public record User(String username, String email, int age) {

    public User {
        if (age < 0) {
            throw new IllegalArgumentException("age must be >= 0");
        }
        if (username == null || username.isBlank()) {
            throw new IllegalArgumentException("username required");
        }
        // compiler adds:
        // this.username = username;
        // this.email = email;
        // this.age = age;
    }
}
```

Use compact constructor when you mostly need validation/normalization.

### 3.3 Extra constructors / factories

You can add overloaded constructors or static factories:

```java
public record Range(int start, int end) {

    public Range {
        if (start > end) {
            throw new IllegalArgumentException("start > end");
        }
    }

    public Range(int size) {
        this(0, size - 1);
    }

    public static Range ofInclusive(int start, int end) {
        return new Range(start, end);
    }
}
```

---

## 4. Methods and interfaces

Records are full classes; you can add methods and implement interfaces.

```java
public record Money(String currency, long cents) {

    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(currency, cents + other.cents);
    }

    public String formatted() {
        return "%s %.2f".formatted(currency, cents / 100.0);
    }
}
```

Implementing interfaces:

```java
public record UserId(long value) implements Comparable<UserId> {

    @Override
    public int compareTo(UserId other) {
        return Long.compare(this.value, other.value);
    }
}
```

---

## 5. Immutability and “withers”

Records are **shallowly immutable**:

* Fields are final.
* Referenced objects (like `List`) can still be mutable.

### 5.1 “With” style methods

To “modify” a record, you create a new instance:

```java
public record User(String username, String email, int age) {

    public User withEmail(String newEmail) {
        return new User(username, newEmail, age);
    }
}

// usage
User u1 = new User("alice", "a@old.com", 25);
User u2 = u1.withEmail("a@new.com"); // new instance
```

You write these manually; there’s no built-in `with` keyword (yet).

### 5.2 Defensive copies for collections

If you want stronger immutability:

```java
public record Tags(List<String> values) {

    public Tags {
        values = List.copyOf(values); // unmodifiable defensive copy
    }
}
```

Without that, callers can still mutate `values()` if it’s a mutable list.

---

## 6. equals, hashCode, toString

Generated **automatically** using **all components**, in order.

```java
record Point(int x, int y) { }

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);

p1.equals(p2);   // true
p1.hashCode();   // same for p1 and p2
p1.toString();   // "Point[x=1, y=2]"
```

You *can* override them, but in most cases you should not; that’s the entire point of using a record.

---

## 7. Inheritance & sealed types

### 7.1 Inheritance limitations

Not allowed:

```java
// ❌ cannot extend another class
public record SpecialUser(String username) extends BaseUser { }
```

Allowed:

```java
public record SpecialUser(String username) implements UserRole { }
```

### 7.2 Sealed interfaces + records

Nice combination for algebraic-style models:

```java
public sealed interface Shape
    permits Circle, Rectangle, Square { }

public record Circle(double radius) implements Shape { }
public record Rectangle(double width, double height) implements Shape { }
public record Square(double side) implements Shape { }
```

---

## 8. Pattern matching with records

Record patterns + sealed types = clean `switch`.

Java 21 style:

```java
sealed interface Shape permits Circle, Rectangle, Square { }
record Circle(double radius) implements Shape { }
record Rectangle(double width, double height) implements Shape { }
record Square(double side) implements Shape { }

double area(Shape shape) {
    return switch (shape) {
        case Circle(double r)              -> Math.PI * r * r;
        case Rectangle(double w, double h) -> w * h;
        case Square(double s)              -> s * s;
    };
}
```

The pattern `Circle(double r)` **deconstructs** the record automatically.

---

## 9. Records vs normal classes

### 9.1 Use a record when

* Primary purpose = data.
* Immutable by design.
* Equality by all fields.
* You would normally generate `equals/hashCode/toString`.

Good fits:

* DTOs (request/response) in APIs.
* Config groups (`Features`, `Limits`).
* Domain value objects (`EmailAddress`, `UserId`, `Money`).
* Command objects (`CreateUserCommand(String name, String email)`).

### 9.2 Use a class when

* You need mutability / builders.
* Framework requires no-arg constructor, proxies (JPA/Hibernate).
* Identity rules are not “all fields”.
* You want classic inheritance from a base class.

---

## 10. Records with generics

Records can be generic:

```java
public record Pair<L, R>(L left, R right) {

    public static <L, R> Pair<L, R> of(L left, R right) {
        return new Pair<>(left, right);
    }
}

// usage
Pair<String, Integer> p = Pair.of("age", 30);
String key   = p.left();
Integer val  = p.right();
```

---

## 11. Local records

You can declare a record **inside a method** for small, local shapes.

```java
public void process() {
    record Entry(String key, int value) { }

    Entry e = new Entry("x", 42);
    System.out.println(e);
}
```

Useful for stream pipelines or temporary result containers in algorithms.

---

## 12. Records & frameworks (Jackson, Spring, JPA)

### 12.1 Jackson

Modern Jackson handles records well:

```java
public record UserResponse(Long id, String name) { }
```

Jackson uses the canonical constructor and parameter names.

If names don’t match JSON fields, use `@JsonProperty`:

```java
public record UserResponse(
    @JsonProperty("id")   Long id,
    @JsonProperty("name") String name
) { }
```

### 12.2 Spring `@ConfigurationProperties`

Records are a clean fit for config binding:

```java
@ConfigurationProperties(prefix = "app")
public record AppProps(
    URI host,
    Features features,
    Limits limits
) {
    public record Features(boolean signup, boolean metrics) { }
    public record Limits(DataSize uploads, Duration timeout) { }
}
```

Spring binds via constructor with component names.

### 12.3 JPA (warning)

Records are generally **not** good JPA entities:

* JPA wants no-arg constructor + mutability + proxies.
* Record semantics fight that model.

Use:

* **Records** for DTOs and value objects.
* **Classes** for entities.

---

## 13. Serialization & performance notes

### 13.1 Java serialization

You can make records `Serializable`:

```java
public record Event(String type, Instant at)
    implements Serializable { }
```

But in modern systems, JSON / protobuf / Avro is usually preferred over Java’s native serialization.

### 13.2 Performance

Records are similar to small final classes:

* No magic performance boost vs normal class.
* Less chance of bugs in equals/hashCode.
* Good as keys in maps/sets because of value semantics.

---

## 14. Mental checklist

Before you create a record, quickly check:

1. Is this type **primarily data**?
2. Should it be **immutable**?
3. Is identity **all fields**?
4. Does it **not** need JPA-style framework magic?
5. Does a generated `equals/hashCode/toString` make sense as-is?

If yes → **record is the default**.
If not sure → start with a record; you can always refactor to a class later if a framework forces you to.

---
