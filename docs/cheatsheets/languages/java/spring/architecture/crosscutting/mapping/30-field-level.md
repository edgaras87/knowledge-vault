---
title: Field-Level Mapping
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - cheatsheet 
  
summary: A comprehensive cheatsheet on MapStruct field-level mapping scenarios, detailing common real-world patterns and providing practical examples for Java Spring applications.
aliases:
    - MapStruct Field-Level Mapping Scenarios Cheatsheet

---

# **MapStruct – Field-Level Mapping Scenarios Cheatsheet**

### *Real-world mapping patterns you will use every day in layered applications*

---

## **Contents (Table Version)**

| Link                                                                      | Description                                         |
|---------------------------------------------------------------------------| --------------------------------------------------- |
| [Flat → Nested Mapping](#1-flat--nested-mapping)                          | Convert flat DTOs into nested domain models.        |
| [Nested → Flat Mapping](#2-nested--flat-mapping)                          | Extract nested domain fields into simple DTOs.      |
| [Two-Level Deep Nested Mapping](#3-two-level-deep-nested-mapping)         | Map deep nested structures across multiple levels.  |
| [Mapping DTO → Command → Domain](#4-mapping-dto--command--domain)         | Typical layered flow from web → app → domain.       |
| [PATCH-Style Updates (ignore nulls)](#5-patchstyle-updates-ignore-nulls)  | Update existing objects while ignoring null fields. |
| [Derived Fields With Expressions](#6-derived-fields-with-expressions)     | Compute fields dynamically using Java expressions.  |
| [Mapping Using Helper Classes](#7-mapping-using-helper-classes)           | Inject helper services into mappings.               |
| [List/Set/Map Mappings](#8-listsetmap-mappings)                           | Convert collections and container types.            |
| [Enum Transformations](#9-enum-transformations)                           | Convert between different enum types.               |
| [Optional Fields Mapping](#10-optional-fields-mapping)                    | Handle Optional values in DTO or domain.            |
| [Record ↔ Class Mapping](#11-record--class-mapping)                       | Map between Java Records and classic POJOs.         |
| [Reusing Mappings With Inheritance](#12-reusing-mappings-with-inheritance) | Share mapping rules across multiple methods.        |

---

## **Sample Data Model (Used in All Examples)**

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

## **1. Flat → Nested Mapping** {#1-flat--nested-mapping}

**What this scenario solves:**
When your incoming DTO is flat (e.g., `"city": "Berlin"`) but your domain model uses nested structures, MapStruct can build the nested object automatically.

### Example

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {

    @Mapping(target = "address.city", source = "city")
    @Mapping(target = "address.country", constant = "DE")
    User toDomain(CreateUserRequest req);
}
```

### Input

```json
{ "name": "Alice", "city": "Berlin" }
```

### Output

```
User {
  name = "Alice",
  address.city = "Berlin",
  address.country = "DE"
}
```

---

## **2. Nested → Flat Mapping** {#2-nested--flat-mapping}

**What this scenario solves:**
Your API responses often need **flat, simple DTOs**, even if the domain model is nested.

### Example

```java
@Mapping(source = "address.city", target = "city")
UserResponse toResponse(User user);
```

### Output

```
UserResponse(id=1, name="Alice", city="Berlin")
```

---

## **3. Two-Level Deep Nested Mapping**

**What this scenario solves:**
Sometimes values sit several levels deep — MapStruct can extract them cleanly.

### Example

```java
@Mapping(target = "region", source = "address.country.region.name")
LocationResponse toLocation(User user);
```

MapStruct creates all needed null checks.

---

## **4. Mapping DTO → Command → Domain** {#4-mapping-dto--command--domain}

**What this scenario solves:**
Layered architecture splits mapping into two steps:

* Web DTO → Command (app layer)
* Command → Domain (business layer)

### Step 1: DTO → Command

```java
@Mapper(config = GlobalMapperConfig.class)
interface UserRequestMapper {
    CreateUserCommand toCommand(CreateUserRequest req);
}
```

### Step 2: Command → Domain

```java
@Mapper(config = GlobalMapperConfig.class)
interface UserCommandMapper {

    @Mapping(target = "address.city", source = "city")
    @Mapping(target = "address.country", constant = "DE")
    User toDomain(CreateUserCommand cmd);
}
```

This avoids leaking HTTP DTOs into deeper layers.

---

## **5. PATCH-Style Updates (ignore nulls)**

**What this scenario solves:**
Allows you to update an existing object **without overwriting fields with null**.

> Works together with `nullValuePropertyMappingStrategy = IGNORE`.

### Config

```java
nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
```

### Mapper

```java
void update(@MappingTarget User user, UpdateUserRequest req);
```

### Example

Input:

```json
{ "city": "Hamburg", "name": null }
```

Original:

```
User{name="Alice", address.city="Berlin"}
```

Result:

```
User{name="Alice", address.city="Hamburg"}
```

Null value is ignored.

---

# **6. Derived Fields With Expressions**

**What this scenario solves:**
Compute new values directly during mapping (slugs, URIs, timestamps, etc.).

### Example — slug creation

```java
@Mapping(
  target = "slug",
  expression = "java(req.name().toLowerCase().replace(\" \", \"-\"))"
)
Category toDomain(CreateCategoryRequest req);
```

Produces:

```
name = "Home Products"
slug = "home-products"
```

---

## **7. Mapping Using Helper Classes**

**What this scenario solves:**
Some values need custom logic or external services — inject helpers into mapping using `@Context`.

### Helper

```java
@Component
public class CountryResolver {
    public String resolve(String city) {
        return city.equals("Berlin") ? "DE" : "UNKNOWN";
    }
}
```

### Mapper

```java
@Mapping(
  target = "address.country",
  expression = "java(resolver.resolve(req.city()))"
)
User toDomain(CreateUserRequest req, @Context CountryResolver resolver);
```

---

## **8. List/Set/Map Mappings**

**What this scenario solves:**
Mapping collections is extremely common — MapStruct does it automatically.

### Lists

```java
List<UserResponse> toResponses(List<User> users);
```

### Sets

```java
Set<AddressDto> toDtos(Set<Address> addresses);
```

### Maps

```java
Map<Long, String> toMap(List<User> users);
```

Requires a mapping method for:

```java
String map(User user)
```

---

## **9. Enum Transformations**

**What this scenario solves:**
Domain enums and API enums often differ.

### Example — identical names (auto)

```java
StatusDto toDto(Status status);
```

### Example — custom mapping

```java
@ValueMappings({
    @ValueMapping(source = "ACTIVE",  target = "ACTIVE"),
    @ValueMapping(source = "BLOCKED", target = "BLOCKED")
})
StatusDto toDto(Status status);
```

---

## **10. Optional Fields Mapping**

**What this scenario solves:**
Sometimes DTO or domain uses `Optional`. MapStruct can unwrap using custom methods.

### Example

```java
default String unwrap(Optional<String> o) {
    return o.orElse(null);
}
```

MapStruct will use it automatically.

---

## **11. Record ↔ Class Mapping** {#11-record--class-mapping}

**What this scenario solves:**
Java Records map cleanly to/from POJOs — perfect for request/response DTOs.

### Example

```java
UserResponse toResponse(User user);
User toDomain(CreateUserRequest req);
```

MapStruct calls record constructors automatically.

---

## **12. Reusing Mappings With Inheritance**

**What this scenario solves:**
Define rules once, reuse them across many methods.

### Example — base method

```java
@Mapping(source = "address.city", target = "city")
UserResponse toFull(User user);
```

### Derived method

```java
@InheritConfiguration(name = "toFull")
@Mapping(target = "name", ignore = true)
UserResponse toCompact(User user);
```

Keeps mappings DRY and consistent.

---

## ✔️ Final Summary

This file gives you:

* every real-world mapping pattern
* clear examples
* input/output demonstrations
* proven patterns for layered apps

Together with File 1 (Global Config) and File 2 (Mapper Annotations), this forms a **complete MapStruct learning and reference system**.
