---
title: "@Mapping" 
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - cheatsheet 
  
summary: A comprehensive cheatsheet on MapStruct's `@Mapping` annotation, detailing its usage and providing real-world examples for Java Spring applications.
aliases:
    - MapStruct Mapping Annotation Cheatsheet

---



# MapStruct – `@Mapping` Annotation Cheatsheet

`@Mapping` tells MapStruct how to copy one field into another.  
It can rename fields, ignore fields, call helper methods, flatten/nest structures, and compute derived values.

Below is a minimal reference with real examples.

---

## Contents

1. [Setup for examples](#1-setup-for-examples)  
2. [Basic: source → target](#2-basic-source--target)  
3. [Renaming fields](#3-renaming-fields)  
4. [Ignoring fields](#4-ignoring-fields)  
5. [Type conversions (built-in)](#5-type-conversions-built-in)  
6. [Type conversions (custom @Named method)](#6-type-conversions-custom-named-method)  
7. [Flattening nested → flat](#7-flattening-nested--flat)  
8. [Nesting flat → nested](#8-nesting-flat--nested)  
9. [Default values](#9-default-values)  
10. [Constant values](#10-constant-values)  
11. [Expression = custom Java snippet](#11-expression--custom-java-snippet)  
12. [qualifiedByName + @Named helpers](#12-qualifiedbyname--named-helpers)  
13. [@Context parameters](#13-context-parameters)  

---

## 1) Setup for examples

A tiny domain model:

```java
// Domain
public class User {
  public Long id;
  public String email;
  public Profile profile;
}

public class Profile {
  public String displayName;
  public String country;
}

// DTO
public record UserDto(Long id, String email, String name, String country) {}
```

A mapper skeleton:

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
  UserDto toDto(User user);
}
```

Every section below shows how `@Mapping` can change the default behavior.

---

## 2) Basic: source → target {#2-basic-source--target}

The simplest form: tell MapStruct “take this input field and put it in this output field.”

```java
@Mapping(source = "profile.displayName", target = "name")
```

This overrides the default matching.
If omitted, MapStruct tries to match fields by identical names.

---

## 3) Renaming fields

When field names differ:

```java
@Mapping(source = "profile.displayName", target = "name")
@Mapping(source = "profile.country",     target = "country")
UserDto toDto(User user);
```

This is the most common beginner case.

---

## 4) Ignoring fields

Skip a target field entirely:

```java
@Mapping(target = "country", ignore = true)
UserDto toDto(User user);
```

Useful when the DTO shouldn’t expose everything.

---

## 5) Type conversions (built-in)

MapStruct transparently converts many types (e.g., `int` ↔ `String`, enums, numeric widening).

Example: profile.country is `"LT"` but DTO expects it as a `String` anyway — nothing special required.

If converting `Instant → String`, MapStruct knows common conversions:

```java
@Mapping(source = "createdAt", target = "createdAt", dateFormat = "yyyy-MM-dd")
```

---

## 6) Type conversions (custom @Named method)

When MapStruct doesn’t know how to convert type A → B, write a small helper:

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

  @Mapping(source = "profile", target = "country", qualifiedByName = "extractCountry")
  UserDto toDto(User user);

  @Named("extractCountry")
  default String extractCountry(Profile p) {
    return p.country.toUpperCase();
  }
}
```

This is one of the cleanest beginner-friendly techniques.

---

## 7) Flattening nested → flat {#7-flattening-nested--flat}

Turn nested objects into flat DTO fields.

```java
@Mapping(source = "profile.displayName", target = "name")
@Mapping(source = "profile.country",     target = "country")
UserDto toDto(User user);
```

MapStruct walks nested properties automatically.

---

## 8) Nesting flat → nested {#8-nesting-flat--nested}

Reverse direction (DTO → domain):

```java
@Mapping(source = "name",    target = "profile.displayName")
@Mapping(source = "country", target = "profile.country")
User toEntity(UserDto dto);
```

MapStruct creates nested objects automatically if needed.

---

## 9) Default values

If the input field is `null`, use the default:

```java
@Mapping(target = "country", defaultValue = "unknown")
```

Great for partially filled input DTOs.

---

## 10) Constant values {#10-constant-values}

Always use the same literal value:

```java
@Mapping(target = "country", constant = "LT")
```

Useful for audit/policy values injected at map-time.

---

## 11) Expression = custom Java snippet {#11-expression--custom-java-snippet}

The powerful but “last resort” option.
MapStruct inserts Java code directly into the generated mapper.

```java
@Mapping(target = "country", expression = "java(user.profile.country().toUpperCase())")
```

Good for one-off transformations, but use `@Named` helpers when possible — cleaner and testable.

---

## 12) qualifiedByName + @Named helpers {#12-qualifiedbyname--named-helpers}

Let a helper method compute complex values.

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

  @Mapping(source = "profile", target = "country", qualifiedByName = "toUpper")
  UserDto toDto(User user);

  @Named("toUpper")
  default String toUpper(Profile p) {
    return p.country.toUpperCase();
  }
}
```

Why use this?

* Cleaner than long expression strings.
* Reusable.
* Testable.

---

## 13) @Context parameters

Pass extra objects to mapping methods (helpers, config, link builders).

```java
@Mapping(target = "country", source = "ctx", qualifiedByName = "ctxCountry")
UserDto toDto(User user, @Context CountryHelper ctx);

@Named("ctxCountry")
default String ctxCountry(CountryHelper ctx) {
  return ctx.defaultCountry();
}
```

Use this when mapping needs information **not present in the source object** (e.g., base URIs, timezone, policy).




### 13.1 Full example of @Context usage





```java
public class User {
  private Long id;
  private String name;

  // getters/setters...
}

public record UserGreetingDto(Long id, String greeting) {}
```

**A simple Spring bean (helper):**

```java
@Component
public class GreetingHelper {

  public String formatGreeting(String name) {
    return "Hello, " + name + "!";
  }
}
```

**Mapper using @Context:**

```java
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface UserMapper {

  // We map the whole User as source to build UserGreetingDto
  @Mapping(target = "greeting", source = "user", qualifiedByName = "makeGreeting")
  UserGreetingDto toGreetingDto(User user, @Context GreetingHelper helper);

  @Named("makeGreeting")
  default String makeGreeting(User user, @Context GreetingHelper helper) {
    // Use the context bean inside the helper method
    return helper.formatGreeting(user.getName());
  }
}
```

**Service wiring everything together:**

```java
@Service
public class UserService {

  private final UserMapper mapper;
  private final GreetingHelper greetingHelper;

  public UserService(UserMapper mapper, GreetingHelper greetingHelper) {
    this.mapper = mapper;
    this.greetingHelper = greetingHelper;
  }

  public UserGreetingDto getUserGreeting(User user) {
    // Pass the Spring bean as @Context parameter
    return mapper.toGreetingDto(user, greetingHelper);
  }
}
```

### 13.2 What @Context actually does here

* `GreetingHelper` is a normal Spring `@Component`.
* The **service** gets both `UserMapper` and `GreetingHelper` injected by Spring.
* The service calls `mapper.toGreetingDto(user, greetingHelper)` – that second argument is the **context**.
* MapStruct:

  * sees `@Context GreetingHelper helper` on the mapper method;
  * also sees `@Context GreetingHelper helper` on the `@Named("makeGreeting")` method;
  * automatically passes the same instance into that helper method.

### 13.3 When to use @Context

Use `@Context` when:

* You need **extra information** that is not part of the source/target types:

  * helpers (`ProblemLinks`, `GreetingHelper`, `CurrencyFormatter`, `TimeZoneResolver`, etc.),
  * request-scoped data (current user, locale),
  * cycle-avoidance or caching context.
* You want the **same object** available in multiple mapping and helper methods, including nested mappings.

Don’t confuse it with Spring DI:

* Spring injects beans into your services/controllers.
* Your code passes those beans into the mapper as `@Context`.
* MapStruct only takes care of threading that parameter through the mapping graph.





---

# Final Summary

* `source` + `target` = core of field-to-field mapping.
* `ignore` = drop a target field.
* `defaultValue` / `constant` = metadata-based mapping.
* `expression` = inline Java (use carefully).
* `qualifiedByName` + `@Named` = recommended for custom logic.
* `@Context` = bring external knowledge into mapping.
* Works with nested structures in both directions.

This cheatsheet gives you everything needed to understand and use `@Mapping` without noise — just update the examples to fit your project’s folders when copying into your vault.

If you extend your mappers later (MapStructConfig, componentModel, injection strategies), this file stays relevant as the core reference.
