---
title: Mapping Zones 
date: 2025-11-17
tags: 
 - architecture
 - mapping
 - mapstruct
 - layered-architecture
 - clean-architecture
 - web-mappers
 - infra-mappers 
  
summary: Explains why web mappers and infra mappers exist in layered architecture, and how MapStruct supports this separation.
aliases:
  - Spring Mapping Zones
---

# **Mapping Zones in Layered Architecture**

### *Why Web Mappers and Infra Mappers Exist (and How MapStruct Fits In)*

---

## Contents

1. [Purpose of This Document](#1-purpose-of-this-document)  
2. [The Core Architectural Principle](#2-the-core-architectural-principle)  
3. [Why Mapping Exists at All](#3-why-mapping-exists-at-all)  
4. [Mapping Zones](#4-mapping-zones)  
    - [Web-Layer Mapping](#41-web-layer-mapping)  
    - [Infra-Layer Mapping](#42-infra-layer-mapping)  
5. [How MapStruct Fits Into This Picture](#5-how-mapstruct-fits-into-this-picture)  
6. [Shared vs Local Mapper Config (GlobalMapperConfig)](#6-shared-vs-local-mapper-config-globalmapperconfig)  
7. [Where Mappers Live in the Project (Package Layout)](#7-where-mappers-live-in-the-project-package-layout)  
8. [Spring vs Plain Java — What’s Framework, What’s Architecture](#8-spring-vs-plain-java--whats-framework-whats-architecture)  
9. [How This Scales When Your Project Grows](#9-how-this-scales-when-your-project-grows)  
10. [Short Summary (Mental Model)](#10-short-summary-mental-model)

---

## **1. Purpose of This Document**

This document explains a simple but powerful idea behind modern backend systems:

> **Different layers speak different languages → each layer needs its own mapping.**

From that principle comes the separation you keep noticing:        
**web mappers** and **infra mappers**.

This is not a Spring trick.          
It’s a clean-architecture rule that MapStruct happens to support beautifully.

---

## **2. The Core Architectural Principle**

Every layered architecture—Clean Architecture, Hexagonal, Onion, classic Layered—follows one old rule:

> **Higher layers depend inward.
> No layer leaks its types outward.**

The moment you decide:

* controllers shouldn’t return JPA entities;
* HTTP DTOs shouldn’t leak into repositories;
* the domain model shouldn’t know about JSON or SQL;

…you’ve accepted layer separation.

Mapping is the glue that maintains this separation.

---

## **3. Why Mapping Exists at All**

Mapping is necessary because:

* HTTP expects one shape.
* Business logic uses another shape.
* Databases/JPA use a third shape.
* External APIs use yet another shape.

Without mapping:

* your domain would become polluted with JPA annotations,
* controllers would expose DB entities,
* clients would depend on internal structures,
* and the system would collapse into a tight, brittle ball of mud.

Mapping keeps each world clean and decoupled.

---

## **4. Mapping Zones**

Architecture creates two main zones where mapping is needed.

---

### **4.1 Web-Layer Mapping**

Lives in the **web** package.

**Responsibilities:**

* Translate **JSON → Request DTO** (Jackson)
* Translate **Request DTO → Command/Input** (MapStruct)
* Translate **Domain → Response DTO** (MapStruct)

**Purpose:**

* Shape data for HTTP
* Hide internal models
* Express exactly what the client should see

**Example:**

```java
@Mapper(config = GlobalMapperConfig.class)
public interface CategoryWebMapper {
    CreateCategoryCommand toCommand(CreateCategoryRequest request);
    CategoryDetailResponse toDetail(Category category);
}
```

Web mappers never touch:

* JPA entities
* database concerns
* persistence DTOs

---

### **4.2 Infra-Layer Mapping**

Lives in the **infra** package.

**Responsibilities:**

* Translate **Domain ↔ JPA entity**
* (sometimes) Domain ↔ external system DTO

**Purpose:**

* Adapt domain to technology
* Keep persistence replaceable
* Prevent JPA models from bleeding upward

**Example:**

```java
@Mapper(config = GlobalMapperConfig.class)
public interface CategoryJpaMapper {
    CategoryJpaEntity toEntity(Category domain);
    Category toDomain(CategoryJpaEntity entity);
}
```

Infra mappers never touch:

* HTTP DTOs
* input commands
* response shapes

---

## **5. How MapStruct Fits Into This Picture**

MapStruct doesn’t create two mapping worlds.
**Your architecture does.**

MapStruct simply automates the boilerplate of translating between the shapes used in each layer.

You could write all the mapping by hand.
You’d still end up with:

* **web mapping functions** living in web
* **infra mapping functions** living in infra

MapStruct just saves your wrists.

---

## **6. Shared vs Local Mapper Config (GlobalMapperConfig)**

Because both mapping zones use MapStruct, you give them shared rules:

```java
@MapperConfig(
  componentModel = "spring",
  unmappedTargetPolicy = ReportingPolicy.ERROR
)
public interface GlobalMapperConfig {}
```

Each mapper then uses:

```java
@Mapper(config = GlobalMapperConfig.class)
interface SomethingMapper {}
```

This ensures:

* same component model
* same strict/lenient policy
* consistent behavior
* zero duplication

Each mapper can still add **local overrides** (e.g. ignore nulls for web but not for infra).

---

## **7. Where Mappers Live In the Project (Package Layout)**

You avoid dependency cycles by placing the shared config above both layers:

```
src/main/java/com/example/app/

  mapping/
    GlobalMapperConfig.java

  web/
    category/
      CategoryWebMapper.java
      dto/
        request/
        response/

  infra/
    jpa/
      category/
        CategoryJpaMapper.java
        CategoryJpaEntity.java
```

This layout scales cleanly whether you have 5 or 50 features.

---

## **8. Spring vs Plain Java — What’s Framework, What’s Architecture**

This is crucial:

* **Layer separation** is pure architecture
  (works in Java, Kotlin, .NET, Go, Node.js, Python).
* **MapStruct** is optional
  (you could write the mapping manually).
* **Spring** only contributes convenience
  (`componentModel = "spring"` → automatic DI).

If Spring disappeared, your mapping structure would look identical.

If MapStruct disappeared, you would write the mapping by hand—but in the same places.

Architecture stays. Tools change.

---

## **9. How This Scales When Your Project Grows**

The separation gives you long-term stability.

You can add:

* new endpoints
* new entities
* new databases
* event messages
* external API integrations
* different response shapes
* multiple tech-layers

…and each layer manages its own mapping without polluting the others.

The system evolves gracefully instead of becoming a spaghetti cathedral.

---

## **10. Short Summary (Mental Model)**

**Web layer speaks HTTP.
Infra layer speaks DB.
Domain layer speaks business.**

Different languages → different translators.
Translators live in their own zones.
Global config keeps them consistent.
MapStruct makes it automated.
Spring just wires them up.

This is an architectural rule, not a framework trick.

