---
title: Mapping Extra Fields (Record DTOs)
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - patterns
    - cheatsheet
summary: A comprehensive guide to mapping extra fields into immutable record DTOs using MapStruct in Java Spring applications.
aliases:
    - MapStruct Immutable Record Extra Fields Mapping Cheatsheet

---

# Mapping Extra Fields into Immutable Record DTOs

Record DTOs are immutable. They have no setters, and therefore **cannot be modified inside `@AfterMapping`** unless you use a builder.  
This pattern shows the clean ways to map **fields that do not come from the domain itself** (links, URIs, localized strings, timestamps) into a record DTO.

Typical examples:

- `problemsUri`  
- `self` / `collection` links  
- `statusTitle` (localization)  
- `createdAt` normalized to a specific clock  
- policy-based values coming from helpers

---

## Contents

1. [Why this pattern exists](#1-why-this-pattern-exists)  
2. [Pattern 1 — Qualified helper method (recommended)](#2-pattern-1--qualified-helper-method-recommended)  
3. [Pattern 2 — Record + Builder + @AfterMapping](#3-pattern-2--record--builder--aftermapping)  
4. [Pattern 3 — Set extra fields at the controller/service edge](#4-pattern-3--set-extra-fields-at-the-controllerservice-edge)  
5. [When to choose which pattern](#5-when-to-choose-which-pattern)  

---

## 1. Why this pattern exists

A record DTO:

```java
public record UserResponse(Long id, String email, String name, URI self) {}
````

is **immutable**.
MapStruct cannot do:

```java
out.setSelf(...); // impossible
```

So whenever your DTO needs extra fields the domain does not provide, you must use one of the three accepted techniques.

---

## 2. Pattern 1 — Qualified helper method (recommended) {#2-pattern-1--qualified-helper-method-recommended}

Use a `@Named` helper that computes the value, and tell MapStruct to call it using `qualifiedByName`.
Pass external helpers (like link builders) via `@Context`.

```java
// DTO
public record UserResponse(Long id, String email, String name, URI problemsUri) {}
```

```java
// Mapper
@Mapper(componentModel = "spring")
public interface UserWebMapper {

  @Mapping(target = "problemsUri",
           source = "links", 
           qualifiedByName = "linksBase")
  UserResponse toResponse(User user, @Context ProblemLinks links);

  @Named("linksBase")
  default URI linksBase(ProblemLinks links) {
    return links.base();
  }
}
```

**Why it works**

* Record stays immutable.
* Mapping stays declarative.
* Policy logic lives clearly in helper methods.
* Safe for large DTOs and repeated patterns.

This is the **cleanest and most future-proof** solution.

---

## 3. Pattern 2 — Record + Builder + @AfterMapping {#3-pattern-2--record--builder--aftermapping}

If you need to compute multiple derived fields or want to modify the DTO during build, use a **Lombok builder**.
MapStruct will target the builder instead of the record.

```java
// DTO
@Builder(toBuilder = true)
public record UserResponse(Long id, String email, String name, URI problemsUri) {}
```

```java
// Mapper
@Mapper(componentModel = "spring")
public interface UserWebMapper {

  UserResponse toResponse(User user, @Context ProblemLinks links);

  @AfterMapping
  default void addLinks(@MappingTarget UserResponse.UserResponseBuilder out,
                        @Context ProblemLinks links) {
    out.problemsUri(links.base());
  }
}
```

**When good**

* You want to update many fields in one place.
* You want post-processing on the builder.
* You are already using Lombok’s builder style.

**Downside:** adds builder ceremony.

---

## 4. Pattern 3 — Set extra fields at the controller/service edge {#4-pattern-3--set-extra-fields-at-the-controllerservice-edge}

When only one small field is needed, sometimes the simplest is best:

```java
var u = service.get(...);

return new UserResponse(
  u.id(),
  u.email(),
  u.name(),
  links.base()
);
```

**When good**

* Minimal number of extra fields.
* DTO construction is trivial.
* You want to avoid mapper overhead.

**Downside:** logic spreads into controllers if used too often.

---

## 5. When to choose which pattern

### Pattern 1 — Qualified helper method

**Use when:**

* DTOs are pure records.
* Mapping is mostly “shape → shape”.
* One or a few extra fields require controlled policy logic.
* You want the mapping to stay declarative.
  **This is the default choice.**

### Pattern 2 — Builder + @AfterMapping

**Use when:**

* You need to compute many derived fields.
* You want to perform multiple adjustments after primary mapping.
* You like the builder style for DTOs.
  **Best for more complex response shaping.**

### Pattern 3 — Set values outside the mapper

**Use when:**

* Only 1–2 extra fields exist.
* You want to keep mappers extremely lean.
* The constructor call is clearer than custom mapping logic.

---

# Summary

Immutable record DTOs cannot be mutated after construction.
To populate external or derived fields, use:

* **Qualified helper methods** (recommended)
* **Builder + @AfterMapping**
* **Controller/service construction**

All three keep mapping predictable, maintainable, and aligned with layered architecture.
