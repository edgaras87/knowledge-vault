---
title: Glossary
date: 2025-11-18
tags: 
    - java
    - spring
    - architecture
    - crosscutting
    - mapping
    - cheatsheet 
  
summary: A quick-reference glossary of MapStruct concepts used in Java Spring applications, explaining their purpose and application.
aliases:
    - MapStruct Glossary Cheatsheet

---

# MapStruct Glossary

A concise reference for all MapStruct concepts used across your layered architecture.  
Each entry explains what it is and why you use it.

---

## Contents

1. [\@Mapper](#mapper)  
2. [\@MapperConfig](#mapperconfig)  
3. [componentModel = "spring"](#componentmodel--spring)  
4. [injectionStrategy = CONSTRUCTOR](#injectionstrategy--constructor)  
5. [unmappedTargetPolicy](#unmappedtargetpolicy)  
6. [typeConversionPolicy](#typeconversionpolicy)  
7. [nullValuePropertyMappingStrategy](#nullvaluepropertymappingstrategy)  
8. [\@Mapping](#mapping)  
9. [source](#source)  
10. [target](#target)  
11. [ignore = true](#ignore--true)  
12. [expression = "java(...)" ](#expression--java)  
13. [constant](#constant)  
14. [defaultValue](#defaultvalue)  
15. [qualifiedByName](#qualifiedbyname)  
16. [\@Named](#named)  
17. [\@Context](#context)  
18. [\@MappingTarget](#mappingtarget)  
19. [\@ValueMapping](#valuemapping)  
20. [\@IterableMapping](#iterablemapping)  
21. [Default methods](#default-methods-in-mappers)  
22. [Static helper methods](#static-helper-methods)  
23. [Builder support](#builder-support)  
24. [Generated files](#generated-files)  
25. [Suggested Navigation](#suggested-navigation)

---

## @Mapper

Marks an interface as a MapStruct mapper.  
MapStruct generates an implementation at compile time.

---

## @MapperConfig

Holds shared configuration for all mappers.  
Used for strictness rules, injection strategy, and null handling.

---

## componentModel = "spring"

Turns the generated mapper into a Spring bean.  
Required when injecting mappers into services or controllers.

---

## injectionStrategy = CONSTRUCTOR

Forces constructor injection for mapper dependencies.  
Improves immutability and testability.

---

## unmappedTargetPolicy

Controls what happens when a target field has no mapping.  
`ERROR` is recommended to catch breaking changes early.

---

## typeConversionPolicy

Controls automatic type conversions.  
`ERROR` ensures you write explicit converters (safer, predictable).

---

## nullValuePropertyMappingStrategy

Defines how nulls behave when mapping property-to-property.  
`IGNORE` prevents nulls from overwriting existing data (ideal for PATCH/update operations).

---

## @Mapping

The main annotation that customizes field mapping: source, target, qualifiers, constants, expressions, defaults.

---

## source

Specifies the input field to read from:

```

source = "profile.displayName"

```

Used when names differ or for nested paths.

---

## target

Specifies the output field to write into:

```
target = "name"
```

---

## ignore = true

Skips mapping for this field entirely.  
Used when DTO hides a domain field.

---

## expression = "java(...)"

Inline Java expression.  
Useful for simple computed values, but avoid large logic blocks.

Example:

```java
@Mapping(target = "slug", expression = "java(src.name().toLowerCase())")
```

---

## constant

Injects a literal string value:

```java
@Mapping(target = "country", constant = "LT")
```

---

## defaultValue

Used when the source is null:

```java
@Mapping(target = "status", defaultValue = "unknown")
```

---

## qualifiedByName

Links to a helper method annotated with `@Named`.
Clean way to inject transformation logic without long expressions.

---

## @Named

Marks a helper method so it can be selected via `qualifiedByName`.

Example:

```java
@Named("toUpper")
default String toUpper(String value) {
  return value.toUpperCase();
}
```

---

## @Context

Passes additional objects (helpers, formatters, link builders, locale, clock) through mapping operations.

Useful when values are not part of the source object.

---

## @MappingTarget

Used for update operations.
Allows mapping *into* an existing object:

```java
void update(UserDto dto, @MappingTarget User entity);
```

---

## @ValueMapping

Custom enum mapping:

```java
@ValueMapping(source = "ENABLED", target = "ACTIVE")
```

Handles mismatched names between domain and external systems.

---

## @IterableMapping

Custom behavior for list/set mapping.
Used to apply qualifiers to all elements of a collection.

Example:

```java
@IterableMapping(qualifiedByName = "toSummary")
List<UserSummary> toSummaries(List<User> users);
```

---

## Default methods in mappers

Helper methods declared `default` inside a mapper.
MapStruct can call them automatically when signatures match.

---

## Static helper methods

Supported for stateless transformations.
Used when logic does not depend on mapper instance or context.

---

## Builder support

MapStruct can write into builders (Lombok or manual).
Essential when mapping into **records** that need extra fields set via `@AfterMapping`.

---

## Generated files

MapStruct generates implementations under:

```
target/generated-sources/annotations/
```

Useful when debugging or verifying what code was produced.

---

## Suggested Navigation

* **MapStruct Starter** — architecture & workflow
* **@Mapping Annotation Cheatsheet** — all field-level options
* **patterns/** — real-world solutions (context mapping, flattening, link building)
* **glossary** — this page (quick lookup)

