---
title: Nested to Flat
date: 2025-11-18
tags: 
  - java
  - spring
  - architecture
  - crosscutting
  - mapping
  - patterns
  - cheatsheet 
  
summary: A comprehensive guide to implementing Nested to Flat Mapping using MapStruct in Java Spring applications.
aliases:
    - MapStruct Nested to Flat Mapping Cheatsheet

---

# Nested to Flat Mapping

Domain models often have **nested structure** (Profile, Address, Settings), while web APIs prefer **flat, convenient DTOs**:

- `user.profile.displayName` → `name`
- `user.address.city` → `city`
- `user.address.countryCode` → `country`

This pattern shows how to flatten nested objects into flat DTOs with MapStruct, while keeping the mapping clear and maintainable.

---

## Contents

1. [Problem: rich domain vs flat API](#1-problem-rich-domain-vs-flat-api)  
2. [Example domain model](#2-example-domain-model)  
3. [Flat DTO design](#3-flat-dto-design)  
4. [Pattern 1 – Simple nested-to-flat mapping](#4-pattern-1--simple-nested-to-flat-mapping)  
5. [Pattern 2 – Partial flattening with reusable nested DTOs](#5-pattern-2--partial-flattening-with-reusable-nested-dtos)  
6. [Pattern 3 – Collections with nested elements](#6-pattern-3--collections-with-nested-elements)  
7. [When to flatten vs keep nested](#7-when-to-flatten-vs-keep-nested)  
8. [Gotchas and guardrails](#8-gotchas-and-guardrails)  

---

## 1. Problem: rich domain vs flat API

Domain wants to model reality:

```text
User
 ├─ Profile
 │   ├─ displayName
 │   └─ language
 └─ Address
     ├─ street
     ├─ city
     └─ countryCode
````

Web clients often want “just give me `name`, `city`, `country`”.

If you expose entities directly, you leak internal structure and make future refactors painful.
If you hand-write flattening everywhere, you repeat the same boilerplate.

MapStruct can centralize this flattening as a pattern.

---

## 2. Example domain model

```java
// domain/user/User.java
public class User {
  private Long id;
  private Profile profile;
  private Address address;

  // getters/setters...
}

// domain/user/Profile.java
public class Profile {
  private String displayName;
  private String language;

  // getters/setters...
}

// domain/user/Address.java
public class Address {
  private String street;
  private String city;
  private String countryCode;

  // getters/setters...
}
```

---

## 3. Flat DTO design 

Design a DTO that represents **what the API needs**, not the domain structure.

```java
// web/user/dto/response/UserDetailResponse.java
public record UserDetailResponse(
    Long id,
    String name,
    String language,
    String street,
    String city,
    String country
) {}
```

Here:

* `name` ← `user.profile.displayName`
* `language` ← `user.profile.language`
* `street` / `city` / `country` ← `user.address.*`

---

## 4. Pattern 1 – Simple nested-to-flat mapping {#4-pattern-1--simple-nested-to-flat-mapping}

When the flattening is straightforward,

Use `source` paths in `@Mapping` to drill into nested structures.

```java
// web/user/UserWebMapper.java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {

  @Mapping(target = "name",     source = "profile.displayName")
  @Mapping(target = "language", source = "profile.language")
  @Mapping(target = "street",   source = "address.street")
  @Mapping(target = "city",     source = "address.city")
  @Mapping(target = "country",  source = "address.countryCode")
  UserDetailResponse toDetail(User user);
}
```

What MapStruct generates is equivalent to:

```java
UserDetailResponse dto = new UserDetailResponse(
    user.getId(),
    user.getProfile() != null ? user.getProfile().getDisplayName() : null,
    user.getProfile() != null ? user.getProfile().getLanguage() : null,
    user.getAddress() != null ? user.getAddress().getStreet() : null,
    user.getAddress() != null ? user.getAddress().getCity() : null,
    user.getAddress() != null ? user.getAddress().getCountryCode() : null
);
```

but you never write that manually.

**When this pattern is enough**

* Nested structure is simple (1–2 levels).
* DTO is clearly flat and stable.
* No heavy transformation logic is needed.

---

## 5. Pattern 2 – Partial flattening with reusable nested DTOs {#5-pattern-2--partial-flattening-with-reusable-nested-dtos}

Sometimes you want:

* a flat DTO for most fields
* but still a nested object for some parts (e.g., address reused across multiple DTOs).

### Nested DTO

```java
// web/common/dto/AddressResponse.java
public record AddressResponse(
    String street,
    String city,
    String country
) {}
```

### User DTO reusing nested address

```java
// web/user/dto/response/UserWithAddressResponse.java
public record UserWithAddressResponse(
    Long id,
    String name,
    String language,
    AddressResponse address
) {}
```

### Mapper using nested mapping

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {

  @Mapping(target = "name",     source = "profile.displayName")
  @Mapping(target = "language", source = "profile.language")
  @Mapping(target = "address",  source = "address")
  UserWithAddressResponse toUserWithAddress(User user);

  // MapStruct will use this for the nested mapping
  AddressResponse toAddressResponse(Address address);
}
```

MapStruct understands that to map `User.address` → `AddressResponse address`, it should call `toAddressResponse(Address)`.

**When to use this**

* You want to flatten “some” structure but keep address reusable across responses.
* You have multiple endpoints returning the same address shape.

---

## 6. Pattern 3 – Collections with nested elements {#6-pattern-3--collections-with-nested-elements}

Flattening in a list is just the same pattern applied to a collection.

### List DTO

```java
// web/user/dto/response/UserListItemResponse.java
public record UserListItemResponse(
    Long id,
    String name,
    String city
) {}
```

### Mapper

```java
@Mapper(config = GlobalMapperConfig.class)
public interface UserWebMapper {

  @Mapping(target = "name", source = "profile.displayName")
  @Mapping(target = "city", source = "address.city")
  UserListItemResponse toListItem(User user);

  List<UserListItemResponse> toListItems(List<User> users);
}
```

MapStruct:

* maps each `User` → `UserListItemResponse`
* then maps the whole `List<User>` → `List<UserListItemResponse>`

No extra configuration needed.

---

## 7. When to flatten vs keep nested

### Flatten when

* Client wants a “simple view” with only a few important fields.
* Nested structure is implementation detail (internal modeling choice).
* DTOs are used in list views, tables, or search results.

Examples:

* `UserListItemResponse` with `name`, `city`, `status`.
* `OrderSummaryResponse` with `orderNumber`, `total`, `currency`.

### Keep nested when

* Nested object is conceptually meaningful to the client (address, price, money, coordinates).
* The nested structure is reused across many DTOs.
* You want to group fields logically in the response.

Examples:

* `address: { street, city, country }`
* `price: { amount, currency }`

You can also **mix**: flatten some fields and keep others grouped.

---

## 8. Gotchas and guardrails

### 1) Don’t mirror domain just because “it’s easier”

Your DTO shape should reflect **client needs**, not entity layout.
Flatten even if domain is nested, when the API is easier to use that way.

### 2) Be explicit in mappings

Always use explicit `@Mapping` when flattening:

```java
@Mapping(target = "name", source = "profile.displayName")
```

instead of relying on implicit matching, so it’s obvious where data comes from.

### 3) Null nested objects

If `user.getProfile()` or `user.getAddress()` can be null, MapStruct will generate null checks.
Consider whether:

* profile/address should always exist (invariant), or
* you need defaults / fallbacks.

For defaults, you can use `defaultValue` or helper methods.

### 4) Keep flattening at the web edge

Flattening is a **web concern**.
Domain and persistence should keep their own structures; mapping layer reshapes them.

### 5) Avoid huge flat DTOs

If a DTO ends up with 20+ fields after flattening, reconsider the API design:

* split into multiple views (summary vs detail)
* reintroduce small nested DTOs where it makes sense (address, money, metadata)

---

# Summary

Nested → flat mapping is about making **client-friendly DTOs** from **rich domain models**:

* Use `source = "nested.path"` in `@Mapping` for simple flattening.
* Introduce reusable nested DTOs where grouping makes sense.
* Apply the same pattern for lists of nested objects.
* Keep flattening logic in web mappers, not in domain or controllers.

Combined with your other patterns (WebMappingContext, Link Mapping, PATCH updates, Immutable Record Extra Fields), this gives you a solid playbook for shaping response data.
