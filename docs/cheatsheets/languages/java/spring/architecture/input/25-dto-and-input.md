---
title: DTOs & Inputs
date: 2025-11-01
tags: 
    - spring
    - architecture
    - best-practices
    - software-architecture
    - dto
summary: Your definitive doctrine for structuring API inputs in Java Spring applications. Learn when and how to separate transport-layer DTOs from application-layer inputs/commands for clean, maintainable code.
aliases:
    - API Input and Command Structuring in Java Spring Cheatsheet
---

# ✅ **Create User Flow Cheatsheet**

*(Spring Boot — Clean layering, upgrade-ready)*

---

## 📂 **Folder Structure**

```
users/
  web/
    UserController.java
    CreateUserRequest.java    (added only when needed)
    UserResponse.java
  app/
    CreateUserInput.java
    UserService.java
  domain/
    User.java
    UserRepository.java
```

* **Always**: `CreateUserInput`, `UserResponse`
* **Only add when tripwire hits**: `CreateUserRequest`

---

## 🎯 **CreateUserInput** (the core input)

> Transport-agnostic, validated, normalized.

```java
// app/CreateUserInput.java
public record CreateUserInput(
    @NotBlank String name,
    @Email @NotBlank String email
) {
    public CreateUserInput {
        if (name != null)  name = name.trim();
        if (email != null) email = email.trim().toLowerCase();
    }
}
```

**Why:** clean, pure, reusable for REST, Kafka, CLI, etc.

---

## 🌐 **Web Controller** (initial simple phase)

> Directly bind `CreateUserInput` until tripwire.

```java
// web/UserController.java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService service;
    private final UserMapper mapper;

    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserInput in) {
        var created = service.create(in);

        var location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.getId())
            .toUri();

        return ResponseEntity.created(location).body(mapper.toResponse(created));
    }
}
```

---

## 👨‍💼 **Service**

```java
// app/UserService.java
@Service
public class UserService {

    private final UserRepository users;
    private final UserMapper mapper;

    @Transactional
    public User create(CreateUserInput in) {

        if (users.existsByEmailIgnoreCase(in.email())) {
            throw new ConflictException("Email already taken");
        }

        var u = mapper.toEntity(in);
        return users.save(u);
    }
}
```

---

## 🧠 **Entity**

```java
// domain/User.java
@Entity
public class User {

    @Id @GeneratedValue
    private UUID id;

    private String name;
    private String email;

    @Version
    private long version;
}
```

---

## 🔁 **Mapping (MapStruct)**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    User toEntity(CreateUserInput in);

    UserResponse toResponse(User user);
}
```

---

## 📤 **Response DTO**

```java
// web/UserResponse.java
public record UserResponse(
    UUID id,
    String name,
    String email,
    long version
) {}
```

---

# 🚧 **Tripwire Upgrade Section**

*(When REST needs JSON quirks OR 2nd input channel arrives)*

Add **Request DTO**:

```java
// web/CreateUserRequest.java
public record CreateUserRequest(
    @JsonProperty("user_name") String name,
    @Email @NotBlank String email
) {}
```

Update controller to map:

```java
@PostMapping
public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req) {
    var in = new CreateUserInput(req.name(), req.email());
    var created = service.create(in);
    return ResponseEntity.ok(mapper.toResponse(created));
}
```

**Service stays untouched** — the win.

---

# 🧠 Mental Reminders Cheat-Box

**Before tripwires**
✔ Input in controller
✔ No DTO
✔ Cleanest flow

**After tripwire**
🔁 `Request → Input` in controller
🧼 Input stays pure
🚫 Never put Jackson into Input
🔒 Service signature does not change

**Tripwires**

* JSON rename (`@JsonProperty`)
* Custom date format
* Multipart
* OpenAPI field docs
* Second input channel (Kafka/CLI)
* Public API stability needed

---

# 🏁 TL;DR Card

| Layer  | Type                | When                                     |
| ------ | ------------------- | ---------------------------------------- |
| web    | `CreateUserRequest` | only after JSON quirks / multi-transport |
| app    | `CreateUserInput`   | always                                   |
| domain | `User`              | always                                   |
| web    | `UserResponse`      | always                                   |

Mapping directions:

```
Request → Input → Domain → Response
```

Never reverse.

