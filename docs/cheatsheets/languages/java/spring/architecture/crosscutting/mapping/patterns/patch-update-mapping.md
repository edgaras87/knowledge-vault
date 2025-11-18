---
title: Domain Update Mapping
date: 2025-11-18
tags: 
  - java
  - spring
  - architecture
  - crosscutting
  - mapping
  - patterns
  - cheatsheet 
  
summary: A comprehensive guide to implementing PATCH DTO to Domain Update Mapping using MapStruct in Java Spring applications.
aliases:
    - MapStruct PATCH DTO to Domain Update Mapping Cheatsheet

---

# PATCH DTO → Domain Update Mapping

PATCH-style updates mean **“only change what the client sends”**, not “replace the whole object”.  
MapStruct can do this reliably by combining:

- a dedicated PATCH DTO  
- `@MappingTarget` to update an existing object  
- `nullValuePropertyMappingStrategy = IGNORE` so nulls do not erase data

This pattern shows how to wire it for domain and JPA entities.

---

## Contents

1. [Problem: partial updates vs full replacements](#1-problem-partial-updates-vs-full-replacements)  
2. [PATCH DTO design](#2-patch-dto-design)  
3. [Mapper configuration for PATCH updates](#3-mapper-configuration-for-patch-updates)  
4. [Domain update mapping example](#4-domain-update-mapping-example)  
5. [JPA entity update mapping example](#5-jpa-entity-update-mapping-example)  
6. [Controller/service flow](#6-controllerservice-flow)  
7. [Gotchas and guardrails](#7-gotchas-and-guardrails)  

---

## 1. Problem: partial updates vs full replacements

**PUT** usually means:  
> “Replace the whole resource with this representation.”

**PATCH** usually means:  
> “Apply these changes; leave everything else as-is.”

Naive mapping approach:

```java
// ❌ Overwrites existing fields with nulls
User updated = mapper.toDomain(patchDto);
```

If `patchDto.name()` is `null`, a naive mapper sets `user.name = null`, which is wrong for PATCH.

We need a mapping that:

* only touches fields present in the PATCH DTO
* ignores nulls (they mean “no change”)
* works with existing domain objects and JPA entities

---

## 2. PATCH DTO design

Use a separate DTO for PATCH operations.
Common approach: **nullable fields** where `null` means “not provided / no change”.

```java
// web/user/dto/request/UserPatchRequest.java
public record UserPatchRequest(
    String email,    // null → don’t change
    String fullName, // null → don’t change
    Boolean active   // null → don’t change
) {}
```

Optional alternative (more explicit, more verbose):

* Wrap fields in `Optional<T>` or a custom `FieldUpdate<T>` type.
* MapStruct can work with it, but that’s a more advanced pattern.
* For now, nullable fields + IGNORE strategy is enough.

---

## 3. Mapper configuration for PATCH updates

Use a mapper with **null-ignore** behavior for properties.

Global config (recommended):

```java
// mapping/GlobalMapperConfig.java
@MapperConfig(
    componentModel = "spring",
    injectionStrategy = InjectionStrategy.CONSTRUCTOR,
    unmappedTargetPolicy = ReportingPolicy.ERROR,
    typeConversionPolicy = ReportingPolicy.ERROR,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface GlobalMapperConfig {}
```

For PATCH mappers, that `IGNORE` is critical:

* Source field is `null` → MapStruct does **not** touch target field.
* Source field has value → MapStruct sets target field.

You can override this per-mapper if needed:

```java
@Mapper(
    config = GlobalMapperConfig.class,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface UserPatchMapper {}
```

---

## 4. Domain update mapping example

Assume domain model:

```java
// domain/user/User.java
public class User {
  private Long id;
  private String email;
  private String fullName;
  private boolean active;

  // getters/setters...
}
```

PATCH DTO:

```java
// web/user/dto/request/UserPatchRequest.java
public record UserPatchRequest(
    String email,
    String fullName,
    Boolean active
) {}
```

Domain mapper for PATCH:

```java
// app/user/UserPatchMapper.java
@Mapper(
    config = GlobalMapperConfig.class,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface UserPatchMapper {

  /**
   * Apply a PATCH DTO onto an existing domain User.
   * Only non-null fields in patch are applied.
   */
  void applyPatch(UserPatchRequest patch, @MappingTarget User user);
}
```

Behavior:

* `patch.email() == null` → `user.email` unchanged
* `patch.fullName() == "New Name"` → `user.fullName` is updated
* `patch.active() == Boolean.FALSE` → `user.active` becomes `false`

MapStruct generates the `if (patch.getX() != null)` checks for you.

---

## 5. JPA entity update mapping example

If you update directly on JPA entities (common in infra layer):

```java
// infra/jpa/user/UserJpaEntity.java
@Entity
public class UserJpaEntity {
  @Id
  private Long id;
  private String email;
  private String fullName;
  private boolean active;

  // getters/setters...
}
```

Mapper:

```java
// infra/jpa/user/UserJpaPatchMapper.java
@Mapper(
    config = GlobalMapperConfig.class,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface UserJpaPatchMapper {

  /**
   * Apply PATCH DTO onto existing JPA entity.
   */
  void applyPatch(UserPatchRequest patch, @MappingTarget UserJpaEntity entity);
}
```

Typical flow:

1. Load entity: `var entity = repository.findById(id).orElseThrow(...);`
2. Call mapper: `jpaPatchMapper.applyPatch(patchDto, entity);`
3. Save: `repository.save(entity);`

---

## 6. Controller/service flow

### Controller

```java
// web/user/UserController.java
@RestController
@RequestMapping("/users")
public class UserController {

  private final UserService service;

  public UserController(UserService service) {
    this.service = service;
  }

  @PatchMapping("/{id}")
  public UserDetailResponse patchUser(
      @PathVariable Long id,
      @RequestBody UserPatchRequest patch
  ) {
    var updated = service.patchUser(id, patch);
    return /* map domain → response DTO via UserWebMapper */;
  }
}
```

### Service (domain-first example)

```java
// app/user/UserService.java
@Service
public class UserService {

  private final UserRepository repository;
  private final UserPatchMapper patchMapper;

  public UserService(UserRepository repository, UserPatchMapper patchMapper) {
    this.repository = repository;
    this.patchMapper = patchMapper;
  }

  public User patchUser(Long id, UserPatchRequest patch) {
    var user = repository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));

    patchMapper.applyPatch(patch, user);

    return repository.save(user);
  }
}
```

You can mirror the same pattern with JPA entities at infra layer if you prefer that style.

---

## 7. Gotchas and guardrails

### 1) “Null” means “no change”, not “set null”

By design in this pattern:

* `null` = **don’t touch**
* non-null = **update field**

If you need “explicitly clear this field” support, you must use a different encoding (e.g., explicit flags or wrapper types).

---

### 2) Do not reuse PUT DTO as PATCH DTO

Separate DTOs:

* `UserUpdateRequest` (for PUT, all required fields)
* `UserPatchRequest` (for PATCH, all nullable fields)

Mixing them leads to weird semantics.

---

### 3) Keep PATCH logic out of the domain model

The domain object should not know about PATCH vs PUT.
All PATCH semantics live in:

* web DTO
* patch mapper
* service method orchestration

Domain sees only “this user is now in this new state”.

---

### 4) Validation is different for PATCH

For PATCH:

* You usually validate only the **fields present**
* Bean Validation for PATCH needs groups or custom logic
* Do not blindly reuse “create/update” validation groups

This pattern focuses only on **mapping**, not validation.

---

### 5) Be strict with unmapped fields

Keep `unmappedTargetPolicy = ERROR` so newly added fields in domain/DTOs don’t silently get ignored.
Update the PATCH mapper when you introduce new fields that should be patchable.

---

# Summary

PATCH mapping is about **partial, non-destructive updates**:

* Design a **PATCH-specific DTO** with nullable fields.
* Use MapStruct with `@MappingTarget` and `nullValuePropertyMappingStrategy = IGNORE`.
* Apply the patch onto an existing domain object or JPA entity.
* Keep PATCH semantics out of the domain, in the web/app layers.

This gives you predictable PATCH behavior, compile-time safety, and clean separation of concerns.

