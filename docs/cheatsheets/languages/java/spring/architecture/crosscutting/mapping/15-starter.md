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

1. [Dependency Setup (Minimal)](#1-dependency-setup-minimal)  
2. [What MapStruct Actually Does](#2-what-mapstruct-actually-does)  
3. [Orientational Folder Layout](#3-orientational-folder-layout)  
4. [Global Config Setup (the 3 core rules)](#4-global-config-setup-the-3-core-rules)  
5. [Web Mappers (HTTP edge)](#5-web-mappers-http-edge)  
6. [Infra Mappers (JPA / external systems)](#6-infra-mappers-jpa--external-systems)  
7. [Updating Existing Entities (`@MappingTarget`)](#7-updating-existing-entities-mappingtarget)  
8. [Expressions & Helper Classes](#8-expressions--helper-classes)  
9. [Enum Mapping](#9-enum-mapping)  
10. [Null Handling](#10-null-handling)  
11. [Collections](#11-collections)  
12. [Nested Mapping](#12-nested-mapping)  
13. [Custom Methods](#13-custom-methods)  
14. [When NOT to Use MapStruct](#14-when-not-to-use-mapstruct)  
15. [Layered Architecture + MapStruct (Summary)](#15-layered-architecture--mapstruct-summary)  

---

## 1. Dependency Setup (Minimal)

MapStruct needs only **two dependencies** to work:

```xml
<!-- API dependency: -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.6.2</version>
</dependency>


<!-- Annotation processor: -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.6.2</version>
    <scope>provided</scope>
</dependency>
```

Optional if you use Lombok: add normal Lombok dependencies.

Annotation processing must be enabled in your IDE/build tool — nothing else required.

---

## 2. What MapStruct Actually Does

MapStruct generates pure Java mapping code at **compile time** using simple interfaces.

It does not:

* use reflection
* perform business logic
* hit databases
* validate input

Its job is strictly **shape → shape transformations**.

---

## 3. Orientational Folder Layout

*(Just an example showing where mapping concepts typically live.)*

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

## 4. Global Config Setup (the 3 core rules)

Global config defines cross-cutting mapping rules.

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

## 5. Web Mappers (HTTP edge)

Convert:

* **Request DTO → Command**
* **Domain → Response DTO**

Example:

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {
    CreateUserCommand toCommand(CreateUserRequest req);
    UserDetailResponse toDetail(User domain);
}
```

No JPA, timestamps, or security logic here.

---

## 6. Infra Mappers (JPA / external systems) {#6-infra-mappers-jpa--external-systems}

Convert:

* **Domain ↔ JPA entity**
* **Domain ↔ external API models**

Example:

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserJpaMapper {
    UserJpaEntity toEntity(User domain);
    User toDomain(UserJpaEntity entity);
}
```

Domain objects stay pure; JPA entities remain persistence-only.

---

## 7. Updating Existing Entities (`@MappingTarget`)

Used when updating a JPA entity in place:

```java
void updateEntity(User domain, @MappingTarget UserJpaEntity entity);
```

MapStruct only updates fields coming from `domain`.
Missing fields → preserved (because of IGNORE strategy).

---

## 8. Expressions & Helper Classes {#8-expressions--helper-classes}

When simple field-to-field mapping isn’t enough.

For derived values:

```java
@Mapping(target = "problemLink",
         expression = "java(problemLinks.type(domain.getSlug()))")
UserResponse toResponse(User domain, @Context ProblemLinks problemLinks);
```

Use default methods for readability:

```java
default URI toSlugLink(String slug, @Context ProblemLinks links) {
    return links.type(slug);
}
```

---

## 9. Enum Mapping

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

## 10. Null Handling

Commonly used settings:

```java
nullValueCheckStrategy = ALWAYS
nullValuePropertyMappingStrategy = IGNORE
nullValueMappingStrategy = RETURN_DEFAULT
```

These are used for PATCH-like operations or DTOs that should not erase data.

---

## 11. Collections

Supported out of the box:

```java
List<UserListItemResponse> toResponses(List<User> users);
```

If element types have mappers, MapStruct chains them automatically.

---

## 12. Nested Mapping

Flattening:

```java
@Mapping(target = "address.city", source = "cityName")
User toUser(UserFlatDto dto);
```

Or keep nested structures if that matches your domain logic better.

---

## 13. Custom Methods

Default and static methods are available for mapping:

```java
default URI toUri(String raw) {
    return URI.create(raw);
}
```

MapStruct calls it automatically when needed.

---

## 14. When NOT to Use MapStruct

Avoid MapStruct for:

* business decisions
* validation
* repository lookups
* dynamic mapping rules
* side-effectful transformations

MapStruct should stay mechanical.

---

## 15. Layered Architecture + MapStruct (Summary) {#15-layered-architecture--mapstruct-summary}

* Web mappers handle request/response
* Infra mappers handle JPA/external
* Domain is pure
* Commands flow down
* Responses flow up
* Global config ensures consistency
* Mapping stays predictable and boring
