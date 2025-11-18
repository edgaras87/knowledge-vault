---
title: Mapper-Level Annotations 
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - cheatsheet 
  
summary: A comprehensive cheatsheet on MapStruct mapper-level annotations, detailing their usage and providing real-world examples for Java Spring applications.
aliases:
  - MapStruct Mapper Annotations Cheatsheet

---

# **MapStruct – Mapper-Level Annotations Cheatsheet**

## *How to control individual mapping behavior inside each mapper*

---

## **Contents (Table Version)**

| Link                                                            | Description                                                        |
| --------------------------------------------------------------- | ------------------------------------------------------------------ |
| [@Mapper](#1-mapper)                                            | Declares an interface as a MapStruct mapper; generator entrypoint. |
| [@Mapping](#2-mapping)                                          | Maps individual fields (rename, constant, ignore, nested paths).   |
| [@Mappings](#3-mappings)                                        | Old syntax for grouping multiple `@Mapping` annotations together.  |
| [@ValueMapping](#4-valuemapping)                                | Defines mapping for a single enum constant.                        |
| [@ValueMappings](#5-valuemappings)                              | Defines enum-to-enum mappings for multiple constants.              |
| [@AfterMapping](#6-aftermapping)                                | Runs custom logic *after* MapStruct finishes mapping.              |
| [@BeforeMapping](#7-beforemapping)                              | Runs logic *before* generated mapping begins.                      |
| [@Context](#8-context)                                          | Injects helper or utility objects into mapping methods.            |
| [@MappingTarget](#9-mappingtarget)                              | Updates an existing object instead of creating a new one.          |
| [@InheritConfiguration](#10-inheritconfiguration)               | Reuses mapping rules defined in another method.                    |
| [@InheritInverseConfiguration](#11-inheritinverseconfiguration) | Automatically creates the reverse mapping for a method.            |
| [uses](#12-uses-importing-other-mappers)                        | Links this mapper to other mappers it depends on.                  |

---

## **Sample Data Model (Used Across Examples)**

```java
public class User {
    Long id;
    String name;
    Address address;
    Status status;
}

public class Address {
    String city;
    String country;
}

public enum Status { ACTIVE, BLOCKED }
public enum StatusDto { ACTIVE, BLOCKED }

public record CreateUserRequest(String name, String city) {}
public record UserResponse(Long id, String name, String city) {}
```

---

## **1. @Mapper**

**What it does:**
Marks an interface as a MapStruct mapper so MapStruct generates the implementation.

### Example

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {
    UserResponse toResponse(User user);
}
```

### Meaning

This tells MapStruct:
**“Generate UserWebMapperImpl with the proper converter code.”**

With `componentModel = "spring"` in your GlobalConfig, the mapper becomes a Spring bean automatically.

---

## **2. @Mapping**

**What it does:**
Maps a single field from source → target, supporting renames, constants, ignores, nested mappings, and expressions.

---

### Example 1 — renaming a field

```java
@Mapping(source = "address.city", target = "city")
UserResponse toResponse(User user);
```

**Input**

```
user.address.city = "Berlin"
```

**Output**

```
city = "Berlin"
```

---

### Example 2 — constant value

```java
@Mapping(target = "address.country", constant = "DE")
```

---

### Example 3 — ignore a field

```java
@Mapping(target = "address", ignore = true)
```

---

### Example 4 — nested → flat

```java
@Mapping(source = "address.city", target = "city")
```

---

### Example 5 — flat → nested

```java
@Mapping(target = "address.city", source = "city")
```

---

### Example 6 — expression

```java
@Mapping(
  target = "slug",
  expression = "java(user.getName().toLowerCase().replace(\" \", \"-\"))"
)
```

---

## **3. @Mappings**

**What it does:**
Legacy container for multiple `@Mapping` annotations.
You no longer need it because `@Mapping` is repeatable, but it still works.

### Example

```java
@Mappings({
    @Mapping(target = "city", source = "address.city"),
    @Mapping(target = "name", source = "name")
})
UserResponse toResponse(User user);
```

---

## **4. @ValueMapping**

**What it does:**
Maps a single enum value to another enum value.

### Example

```java
@ValueMapping(source = "ACTIVE", target = "ACTIVE")
StatusDto toDto(Status status);
```

---

## **5. @ValueMappings**

**What it does:**
Defines multiple enum mappings at once.

### Example

```java
@ValueMappings({
    @ValueMapping(source = "ACTIVE",  target = "ACTIVE"),
    @ValueMapping(source = "BLOCKED", target = "BLOCKED")
})
StatusDto toDto(Status status);
```

---

## **6. @AfterMapping**

**What it does:**
Runs additional logic *after* MapStruct completes field mapping.
Used for calculated fields, validation, fixing nested data, applying defaults.

### Example

```java
@AfterMapping
default void addCountry(@MappingTarget User user) {
    if (user.getAddress() != null && user.getAddress().getCountry() == null) {
        user.getAddress().setCountry("DE");
    }
}
```

---

## **7. @BeforeMapping**

**What it does:**
Runs custom logic *before* the generated mapping starts.
Useful when preparing the target object or context.

### Example

```java
@BeforeMapping
default void initAddress(@MappingTarget User user) {
    if (user.getAddress() == null) {
        user.setAddress(new Address());
    }
}
```

Rare but helpful in certain scenarios.

---

## **8. @Context**

**What it does:**
Allows you to inject extra helper/utility/service objects into your mapper methods.

### Example

Helper:

```java
@Component
public class CountryResolver {
    public String resolve(String city) {
        return city.equals("Berlin") ? "DE" : "UNKNOWN";
    }
}
```

Mapper:

```java
@Mapping(
  target = "address.country",
  expression = "java(countryResolver.resolve(req.city()))"
)
User toDomain(CreateUserRequest req, @Context CountryResolver countryResolver);
```

---

## **9. @MappingTarget**

**What it does:**
Marks a parameter as the object to UPDATE instead of creating a new one.

This is how you implement PATCH or entity updates.

### Example

```java
void updateEntity(User source, @MappingTarget User target);
```

**Behavior:**

* Null fields are ignored if your GlobalConfig uses `IGNORE`.
* Only provided fields overwrite the target.

---

## **10. @InheritConfiguration**

**What it does:**
Reuses mapping rules from another method — great for consistency.

### Example

Base mapping:

```java
@Mapping(source = "address.city", target = "city")
UserResponse mapFull(User user);
```

Derived mapping:

```java
@InheritConfiguration(name = "mapFull")
@Mapping(target = "id", ignore = true)
UserResponse mapShort(User user);
```

---

## **11. @InheritInverseConfiguration**

**What it does:**
Generates the reverse mapping based on an existing mapping.

### Example

Forward mapping:

```java
@Mapping(target = "address.city", source = "city")
User toDomain(CreateUserRequest req);
```

Reverse mapping:

```java
@InheritInverseConfiguration(name = "toDomain")
CreateUserRequest toRequest(User user);
```

MapStruct builds the reverse rule automatically.

---

## **12. uses (importing other mappers)**

**What it does:**
Lets one mapper use other mappers to handle nested or complex structures.

### Example

```java
@Mapper(
  config = GlobalMapperConfig.class,
  uses = { AddressMapper.class }
)
public interface UserWebMapper {
    @Mapping(target = "city", source = "address")
    UserResponse toResponse(User user);
}
```

MapStruct automatically calls:

```
AddressMapper.map(Address) → String
```

when needed.

---

## ✔️ Final Summary

This file covers all mapper-level behaviors:

* defining mapper interfaces
* mapping rules
* inverse mappings
* post-processing logic
* injecting helpers
* updating existing objects
* delegating to other mappers

These annotations form the **core vocabulary** you use when writing MapStruct interfaces.

---
