---
title: Starter
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - cheatsheet 
  
summary: A practical cheatsheet on using MapStruct in a layered architecture, detailing best practices and providing real-world examples for Java Spring applications.
aliases:
  - MapStruct Layered Architecture Cheatsheet
---


# MapStruct — Practical Cheatsheet for Layered Architecture

MapStruct is a compile-time Java mapper generator. It creates fast, type-safe mapping classes without runtime reflection.  
Used correctly, it keeps the web layer, domain layer, and persistence layer clean and decoupled.

---

## Contents

1. [What MapStruct Actually Does](#1-what-mapstruct-actually-does)  
2. [Orientational Folder Layout](#2-orientational-folder-layout)  
3. [Global Config Setup (the 3 core rules)](#3-global-config-setup-the-3-core-rules)  
4. [Web Mappers (HTTP edge)](#4-web-mappers-http-edge)  
5. [Infra Mappers (JPA--external-systems)](#5-infra-mappers-jpa--external-systems)  
6. [Updating Existing Entities (`@MappingTarget`)](#6-updating-existing-entities-mappingtarget)  
7. [Expressions & Helper Classes](#7-expressions--helper-classes)  
8. [Enum Mapping](#8-enum-mapping)  
9. [Null Handling](#9-null-handling)  
10. [Collections](#10-collections)  
11. [Nested Mapping](#11-nested-mapping)  
12. [Custom Methods](#12-custom-methods)  
13. [When NOT to Use MapStruct](#13-when-not-to-use-mapstruct)  
14. [Layered Architecture + MapStruct (Summary)](#14-layered-architecture--mapstruct-summary)  

---

## 1. What MapStruct Actually Does

MapStruct generates pure Java mapping code at **compile time** based on simple interfaces.

It does not:

- run reflection  
- hit databases  
- perform validation  
- execute business logic

It only transforms shapes → shapes.

---

## 2. Orientational Folder Layout

*(This is only an example. Do not treat it as strict rules; it is here just to show where concepts usually live.)*

```text
src/main/java/com/edge/shopping_cart
├─ mapping/
│  └─ GlobalMapperConfig.java
│
├─ domain/
│  └─ category/
│     └─ Category.java
│
├─ infra/
│  └─ jpa/
│     └─ category/
│        ├─ CategoryJpaEntity.java
│        ├─ CategoryJpaMapper.java
│        └─ CategoryJpaRepository.java
│
├─ app/
│  └─ category/
│     ├─ CreateCategoryCommand.java
│     └─ CategoryService.java
│
└─ web/
   └─ category/
      ├─ CategoryController.java
      ├─ CategoryWebMapper.java
      └─ dto/
         ├─ request/
         │  └─ CreateCategoryRequest.java
         └─ response/
            ├─ CategoryDetailResponse.java
            └─ CategoryListItemResponse.java
```

---

## 3. Global Config Setup (the 3 core rules)

This global config defines cross-cutting mapper behavior across all layers.

```java
@MapperConfig(
    componentModel = "spring",
    injectionStrategy = InjectionStrategy.CONSTRUCTOR,
    unmappedTargetPolicy = ReportingPolicy.ERROR,
    typeConversionPolicy = ReportingPolicy.ERROR,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface GlobalMapperConfig {}
```

### The three most important settings (with explanations)

#### 1) `unmappedTargetPolicy = ReportingPolicy.ERROR`

Tells MapStruct:

> “If you forget to map a field → fail the build.”

This catches DTO/domain drift instantly — extremely useful when refactoring.

#### 2) `typeConversionPolicy = ReportingPolicy.ERROR`

Prevents MapStruct from guessing weird type conversions.
If conversion isn’t obvious (e.g., `String → UUID`), you get a compile-time error — making transformations explicit and safe.

#### 3) `nullValuePropertyMappingStrategy = IGNORE`

Used heavily for *update* mappings.
If a source field is null, the target field is untouched instead of overwritten.
Perfect for PATCH-like DTOs and JPA entity updates.

---

Mappers use the config like this:

```java
@Mapper(config = GlobalMapperConfig.class)
public interface SomethingMapper {}
```

---

## 4. Web Mappers (HTTP edge)

Translates:

**Request DTO → Command**
**Domain → Response DTO**

Example:

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {
    CreateUserCommand toCommand(CreateUserRequest req);
    UserDetailResponse toDetail(User domain);
}
```

These mappers never touch persistence, timestamps, security, or problem handling logic.

---

## 5. Infra Mappers (JPA / external systems) {#5-infra-mappers-jpa--external-systems}

Responsible for moving between **domain** and **infrastructure-specific** shapes.

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserJpaMapper {
    UserJpaEntity toEntity(User domain);
    User toDomain(UserJpaEntity entity);
}
```

Domain objects stay pure; JPA entities remain persistence-only.

---

## 6. Updating Existing Entities (`@MappingTarget`)

Used when updating a JPA entity in place:

```java
void updateEntity(User domain, @MappingTarget UserJpaEntity entity);
```

MapStruct only updates fields coming from `domain`.
Missing fields → preserved (because of IGNORE strategy).

---

## 7. Expressions & Helper Classes {#7-expressions--helper-classes}

When simple field-to-field mapping isn’t enough.

For derived values:

```java
@Mapping(target = "problemsUri",
         expression = "java(problemLinks.type(src.getSlug()))")
UserResponse toResponse(User src, @Context ProblemLinks problemLinks);
```

Use default methods for readability:

```java
default URI toSlugLink(String slug, @Context ProblemLinks links) {
    return links.type(slug);
}
```

---

## 8. Enum Mapping

Automatic when names match.

Manual when they differ:

```java
@ValueMappings({
    @ValueMapping(source = "ENABLED", target = "ACTIVE"),
    @ValueMapping(source = "DISABLED", target = "BLOCKED")
})
StatusDto map(Status domain);
```

---

## 9. Null Handling

Most common setups:

```java
nullValueCheckStrategy = ALWAYS
nullValuePropertyMappingStrategy = IGNORE
nullValueMappingStrategy = RETURN_DEFAULT
```

These are used for PATCH-like operations or DTOs that should not erase data.

---

## 10. Collections

Automatically supported:

```java
List<UserListItemResponse> toListItems(List<User> users);
```

If element types have mappers, MapStruct chains them automatically.

---

## 11. Nested Mapping

Flattening:

```java
@Mapping(target = "address.city", source = "cityName")
User toUser(UserFlatDto dto);
```

For large nested structures, DTOs that mirror the domain are cleaner.

---

## 12. Custom Methods

Every `default` or `static` method is eligible:

```java
default URI toUri(String raw) {
    return URI.create(raw);
}
```

MapStruct calls it automatically when needed.

---

## 13. When NOT to Use MapStruct

Avoid MapStruct when:

* mapping requires business logic or decisions
* mapping depends on repositories or side effects
* mapping rules change dynamically
* validation is necessary during transformation

MapStruct should stay mechanical.

---

## 14. Layered Architecture + MapStruct (Summary)

* Web mappers ←→ handle request/response shapes
* Infra mappers ←→ deal with JPA/external systems
* Domain is pure
* Commands/Inputs flow one way
* Responses flow the other
* Global config keeps consistency
* MapStruct keeps mapping boring and predictable










