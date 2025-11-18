---
title: Global Config
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - cheatsheet 
  
summary: A comprehensive cheatsheet on MapStruct global configuration annotations, detailing their usage and providing real-world examples for Java Spring applications.
aliases:
  - MapStruct Global Config Cheatsheet

---

# **MapStruct – Global Configuration Cheatsheet**

## *How to control MapStruct’s behavior across your entire project*

---

## **Contents (Table Version)**

| Link                                                                    | Description                                                                  |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| [@MapperConfig](#1-mapperconfig)                                        | Defines a shared configuration that all mappers inherit.                     |
| [componentModel](#2-componentmodel)                                     | Controls how mapper instances are created (Spring, default, CDI).            |
| [unmappedTargetPolicy](#3-unmappedtargetpolicy)                         | Determines whether missing fields cause errors, warnings, or are ignored.    |
| [typeConversionPolicy](#4-typeconversionpolicy)                         | Controls whether MapStruct can automatically convert types.                  |
| [injectionStrategy](#5-injectionstrategy)                               | Defines if dependent mappers are injected via constructor, field, or setter. |
| [nullValueCheckStrategy](#6-nullvaluecheckstrategy)                     | Controls if null checks are generated to prevent NPE during mapping.         |
| [nullValuePropertyMappingStrategy](#7-nullvaluepropertymappingstrategy) | Defines how null source properties affect the mapped object.                 |
| [nullValueMappingStrategy](#8-nullvaluemappingstrategy)                 | Behavior when the *entire* source object is null.                            |
| [mappingInheritanceStrategy](#9-mappinginheritancestrategy)             | Allows base mapping rules to be shared/reused across methods.                |
| [builder](#10-builder)                                                  | Configures how MapStruct handles builder-based classes (e.g., Lombok).       |

---

## **Sample Data Model (Used in All Examples)**

```java
public class User {
    Long id;
    String name;
    Address address;
}

public class Address {
    String city;
    String country;
}

public record UserRequest(String name, String city) {}
public record UserResponse(Long id, String name, String city) {}
```

---

## **1. @MapperConfig**

**What it does:**
Defines a *shared, central rulebook* for all mappers — every mapper that references this config inherits consistent behavior.

### Example

```java
@MapperConfig(
    componentModel = "spring",
    unmappedTargetPolicy = ReportingPolicy.ERROR
)
public interface GlobalMapperConfig {}
```

All mappers using:

```java
@Mapper(config = GlobalMapperConfig.class)
```

automatically obey these rules.

**Why it matters:**
Large projects get dozens of mappers — one shared config keeps them predictable and strict.

---

## **2. componentModel**

**What it does:**
Controls how MapStruct creates mapper instances — as a Spring bean, a regular Java class, or CDI-managed bean.

### Example (Spring bean)

```java
@MapperConfig(componentModel = "spring")
public interface GlobalMapperConfig {}
```

### Usage

```java
@Service
public class UserService {
    private final UserWebMapper mapper;

    public UserService(UserWebMapper mapper) {  // constructor-injected
        this.mapper = mapper;
    }
}
```

This works because `componentModel = "spring"` turns the mapper into a Spring-managed bean.

---

## **3. unmappedTargetPolicy**

**What it does:**
Controls whether MapStruct complains when a target field isn’t explicitly mapped.

### Example

```java
@MapperConfig(unmappedTargetPolicy = ReportingPolicy.ERROR)
```

### Why it's useful

If you add:

```java
String nickname;
```

to your domain object but forget to update the mapper, your build fails with:

```
ERROR: Unmapped target property: "nickname"
```

You catch mapping bugs immediately.

---

## **4. typeConversionPolicy**

**What it does:**
Defines whether MapStruct is allowed to perform automatic type conversions (e.g., `int → String`).

### Example

```java
@MapperConfig(typeConversionPolicy = ReportingPolicy.ERROR)
```

### Prevents accidental conversions

Without this policy:

```java
27 → "27"
```

could happen silently.

With the policy:
You must write an explicit expression:

```java
@Mapping(target = "age", expression = "java(String.valueOf(req.age()))")
```

This avoids invisible bugs.

---

## **5. injectionStrategy**

**What it does:**
Controls how dependent mappers are injected — constructor injection (best), field injection, or setter injection.

### Example (constructor injection)

```java
@MapperConfig(injectionStrategy = InjectionStrategy.CONSTRUCTOR)
```

MapStruct then generates:

```java
public UserWebMapperImpl(AddressMapper addressMapper) {
    this.addressMapper = addressMapper;
}
```

Constructor injection is predictable and works perfectly with Spring Boot.

---

## **6. nullValueCheckStrategy**

**What it does:**
Enforces when MapStruct should add `null` checks to prevent NullPointerExceptions in nested mappings.

### Example

```java
@MapperConfig(nullValueCheckStrategy = NullValueCheckStrategy.ALWAYS)
```

Generated mapping:

```java
if (user.getAddress() != null) {
    response.setCity(user.getAddress().getCity());
}
```

Without this:
MapStruct may assume non-null and cause runtime NPEs.

---

## **7. nullValuePropertyMappingStrategy**

**What it does:**
Defines what happens when *individual fields* in the source object are null.

### Example (ignore nulls)

```java
@MapperConfig(
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
```

### PATCH-like update behavior

Input:

```json
{ "name": null, "city": "Hamburg" }
```

Original entity:

```
User{name="Alice", address.city="Berlin"}
```

After mapping:

```
User{name="Alice", address.city="Hamburg"}
```

Null name is ignored, city is updated.

---

## **8. nullValueMappingStrategy**

**What it does:**
Controls what mapping produces when the *entire source object* is null.

### Example

```java
@MapperConfig(nullValueMappingStrategy = NullValueMappingStrategy.RETURN_DEFAULT)
```

Mapping a null list returns an empty list, not `null`.

### Benefit

You avoid endless `null` checks in collection mappings.

---

## **9. mappingInheritanceStrategy**

**What it does:**
Allows you to define “base rules” once and have them automatically applied to other methods.

### Example

```java
@MapperConfig(
  mappingInheritanceStrategy = MappingInheritanceStrategy.AUTO_INHERIT_FROM_CONFIG
)
public interface GlobalMapperConfig {

    @Mapping(target = "name", expression = "java(src.getName().trim())")
    User base(User src);
}
```

Any mapper using this config inherits the trimming behavior.

---

## **10. builder**

**What it does:**
Controls how MapStruct handles builder-pattern objects (e.g., Lombok @Builder).

### Example

Domain:

```java
@Builder
public class User {
    Long id;
    String name;
}
```

Config:

```java
@MapperConfig(builder = @Builder(disableBuilder = false))
```

MapStruct uses:

```
User.builder().name(req.name()).build();
```

If you turn builders off:

```java
disableBuilder = true
```

→ MapStruct falls back to setters or constructor.

---

## ✔️ Final Summary

Global config defines:

* mapper creation style
* strictness policies
* null handling
* builder behavior
* inheritance rules

It is literally **the project-wide mapping rulebook**.
Every mapper inherits it unless you explicitly override defaults.

