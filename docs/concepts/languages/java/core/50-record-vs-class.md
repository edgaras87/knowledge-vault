---
title: Using Records vs Classes 
date: 2025-11-14
tags: 
    - java
    - spring
    - records
    - classes 
  
summary: When to use Java records vs classes for Spring components like @Component, @Service, or @ConfigurationProperties.
aliases:
  - Spring Records vs Classes
---


# Using Records vs Classes for Spring Components

When to use Java `record` vs `class` for Spring components like `@Component`, `@Service`, or `@ConfigurationProperties`?

---

## 1. What records are actually for

Records are:

* final classes
* shallow, immutable data carriers
* ideal for: DTOs, props, “this is just a bundle of values”

That’s why they are great for things like:

```java
@ConfigurationProperties(prefix = "app")
public record AppProps(
    URI host,
    Duration timeout
) {}
```

Here `AppProps` **is** the config data. The whole type *is defined by its state*.

---

## 2. What `DataSourceInfo` is

Look at this again:

```java
@Component
public class DataSourceInfo {

    private final DataSourceProperties props;

    public DataSourceInfo(DataSourceProperties props) {
        this.props = props;
    }

    public String url() {
        return props.getUrl();
    }

    public String username() {
        return props.getUsername();
    }
}
```

This thing is not “I am data”.
This thing is “I **use** another bean and expose some helper behavior”.

Key points:

* It **wraps** `DataSourceProperties` (which is the real data holder).
* It’s basically a **service/helper**, not a POJO representing state.
* Today it has just two methods, tomorrow you might add:

  * logging
  * caching
  * conditional logic
  * derived values

Semantically, that’s closer to a normal service class.

You *could* write it as a record:

```java
@Component
public record DataSourceInfo(DataSourceProperties props) {

    public String url() {
        return props.getUrl();
    }

    public String username() {
        return props.getUsername();
    }
}
```

This will work.

But then you’ve turned a behavior-oriented Spring bean into a “looks-like-data” type. That’s a bit misleading.

---

## 3. Practical reasons to prefer classes for components

There are also some practical angles:

1. **Records are final classes**

    * Final classes can’t be subclassed.
    * Some proxy mechanisms (CGLIB) rely on subclassing.
    * If one day you put AOP on that bean (logging, metrics, transactions) and Spring wants to use class-based proxying, final can bite you.
    * Not guaranteed problem in every case, but it’s a landmine.

2. **Semantics / mental model**

    * `record` → “this *is* a value object”
    * `class` → “this *does* something”
    * Components/services usually fall into the second bucket.

3. **Evolution**

    * A helper like `DataSourceInfo` is likely to grow behavior: formatting, edge cases, extra methods.
    * A record is meant to stay a simple data carrier; you *can* add behavior, but it’s not what most readers expect.

---

## 4. Where records shine vs where classes shine in your setup

In **your style** of project:

**Use records for:**

* `@ConfigurationProperties` (`AppProps`, `SecurityProps`, `MailProps`, `TimeProps`, …)
* DTOs (`UserDetailResponse`, `CreateUserRequest`, command/input types)
* Simple value objects with no behavior

**Use classes for:**

* `@Component` / `@Service` / `@Repository` beans
* Helpers wrapping env, properties, clocks, message sources:

    * `TimeFacade`
    * `ProblemLinks`
    * `Messages`
    * `SpringEnvSupport`
    * `ServerInfo`
* Anything where behavior is the main thing and state is just implementation detail

So the examples I showed as `class` are in the “helper/service” bucket, not “data carrier” bucket.

---

## 5. Brutally honest rule-of-thumb

* If the type’s *meaning* is “this is a structured piece of data” → **record**.
* If the type’s *meaning* is “this thing collaborates with other beans to **do stuff**” → **class**.

`DataSourceInfo` lives in the second group. That’s why I wrote it as a class, not because records are broken, but because the semantics and future-proofing are better with a class.

As you keep building your project, you can almost draw a line:

* `config.props`, `web.dto`, etc. → mostly records
* `web.support`, `app`, `infra`, `i18n`, `time` helpers → mostly classes

That separation will make your codebase feel very intentional instead of “records everywhere because they’re shiny”.
