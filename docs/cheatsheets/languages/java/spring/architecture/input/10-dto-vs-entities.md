---
title: DTOs vs Entities 
date: 2025-10-22
tags:
    - spring
    - dto
    - entity
    - jpa
summary: Distinguish between persistence entities and API DTOs in Spring Boot applications. Best practices, mapping strategies, and pitfalls.
aliases:
  - Spring Boot — DTOs vs Entities Cheatsheet
---


# 🧩 Spring Boot — DTOs vs Entities Cheatsheet

> **Essence:** Entities are your *persistence model* (database schema).  
> DTOs (Data Transfer Objects) are your *API model* (request/response contracts).  
> Mixing them leads to fragile APIs, security leaks, and serialization pain.

---

## 🔑 Core Distinction

| Aspect | Entity | DTO |
|:--|:--|:--|
| **Purpose** | Persistence (DB schema) | API request/response |
| **Managed by** | JPA/Hibernate | Jackson / Validation |
| **Contains** | IDs, relationships, DB constraints | only fields relevant to API |
| **Annotations** | `@Entity`, `@Table`, `@Column` | `@NotBlank`, `@Email`, etc. |
| **Visibility** | Internal | Public |
| **Evolution** | DB-driven | Client-driven |

> **Remember:** Jackson annotations ≠ DTOs. They’re formatting tools, not boundary definitions.

---

## 🚫 When You *Can* Skip DTOs

You can temporarily skip DTOs if:
- The app is small and internal.
- Entities are flat (no relations, no sensitive fields).
- You accept API/DB coupling risk.

But even then:
- Use `@JsonIgnore` for sensitive data.
- Handle cycles (`@JsonManagedReference` / `@JsonBackReference`).
- Expect refactoring later when complexity grows.

---

## ✅ Why You Should Use DTOs

**Security:** Prevent leaking internal fields (e.g., `password`, `isAdmin`, audit data).  
**Stability:** Entities evolve freely while DTOs keep a stable public API.  
**Validation:** Add `@NotBlank`, `@Email`, etc. for request validation.  
**Flexibility:** Tailor different DTOs per endpoint (don’t reuse everything).  
**Performance:** Avoid lazy-loading cycles or massive entity graphs.  
**Versioning:** Create `v2` DTOs without breaking clients.

---

## 🧱 Example: Contact Entity + DTOs

```java
@Entity
@Table(name = "contacts")
public class Contact {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private String id;

  private String name;
  private String email;
  private String photoUrl;

  // Never expose this!
  private String password;
}
```

### Request DTOs

```java
public record CreateContactRequest(
    @NotBlank String name,
    @Email String email,
    String photoUrl,
    @NotBlank String password
) {}

public record UpdateContactRequest(
    String name,
    @Email String email,
    String photoUrl,
    String password
) {}
```

### Response DTO

```java
public record ContactResponse(
    String id,
    String name,
    String email,
    String photoUrl
) {}
```

---

## 🔁 Mapping Strategies (Entity ↔ DTO)

Mapping = the glue between persistence and API layers.
Several approaches exist — pick one main strategy.

### 1. **MapStruct (recommended for production)**

*Fast, compile-time, explicit.*

```java
@Mapper(componentModel = "spring")
public interface ContactMapper {

  @Mapping(target = "id", ignore = true)
  Contact toEntity(CreateContactRequest req);

  @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
  void updateEntity(UpdateContactRequest req, @MappingTarget Contact target);

  ContactResponse toResponse(Contact entity);
}
```

**Service example:**

```java
@Service
@RequiredArgsConstructor
public class ContactService {
  private final ContactRepository repo;
  private final ContactMapper mapper;
  private final PasswordEncoder encoder;

  public ContactResponse create(CreateContactRequest req) {
    Contact c = mapper.toEntity(req);
    c.setPassword(encoder.encode(req.password()));
    return mapper.toResponse(repo.save(c));
  }

  public ContactResponse update(String id, UpdateContactRequest req) {
    Contact c = repo.findById(id).orElseThrow();
    mapper.updateEntity(req, c);
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(encoder.encode(req.password()));
    }
    return mapper.toResponse(repo.save(c));
  }
}
```

---

### 2. **ModelMapper (fast prototyping)**

*Reflection-based, minimal setup, slower.*

```java
@Configuration
public class ModelMapperConfig {
  @Bean
  public ModelMapper modelMapper() {
    ModelMapper mm = new ModelMapper();
    mm.getConfiguration().setPropertyCondition(Conditions.isNotNull());
    return mm;
  }
}
```

Usage:

```java
Contact c = modelMapper.map(req, Contact.class);
ContactResponse dto = modelMapper.map(entity, ContactResponse.class);
```

Pros: zero boilerplate.
Cons: less control, easy to miss subtle mismatches.

---

### 3. **Manual Mapping (small apps)**

*Ultimate control, no dependencies.*

```java
@Component
public class ContactMapper {

  public Contact toEntity(CreateContactRequest req) {
    Contact c = new Contact();
    c.setName(req.name());
    c.setEmail(req.email());
    c.setPhotoUrl(req.photoUrl());
    c.setPassword(req.password());
    return c;
  }

  public void applyUpdate(UpdateContactRequest req, Contact c) {
    if (req.name() != null) c.setName(req.name());
    if (req.email() != null) c.setEmail(req.email());
    if (req.photoUrl() != null) c.setPhotoUrl(req.photoUrl());
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(req.password());
    }
  }

  public ContactResponse toResponse(Contact c) {
    return new ContactResponse(c.getId(), c.getName(), c.getEmail(), c.getPhotoUrl());
  }
}
```

Pros: no magic.
Cons: verbose as app grows.

---

## 🧠 Spring Data Interface Projections

*Read-only, auto-mapped “DTOs” generated by Spring Data.*

```java
public interface ContactView {
  String getId();
  String getName();
}

public interface ContactRepository extends JpaRepository<Contact, String> {
  List<ContactView> findByNameContainingIgnoreCase(String q);
}
```

Controller:

```java
@GetMapping("/contacts")
public List<ContactView> list(@RequestParam String q) {
  return repo.findByNameContainingIgnoreCase(q);
}
```

✅ Great for lightweight reads.
❌ Not for writes (can’t persist).

---

## 🔄 Serialization Cycles (Jackson)

When you serialize bidirectional relationships (`User` ↔ `Order`), Jackson can loop infinitely.

### Option 1: `@JsonManagedReference` / `@JsonBackReference`

Breaks recursion by ignoring the “back” side during serialization.

```java
class User {
  @JsonManagedReference List<Order> orders;
}

class Order {
  @JsonBackReference User user;
}
```

Good for simple parent→child relationships.

### Option 2: `@JsonIdentityInfo`

Uses object IDs instead of nesting endlessly.

```java
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
class User { Long id; List<Order> orders; }

@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
class Order { Long id; User user; }
```

Good for complex graphs or both-direction references.

---

## ⚙️ Validation: API vs DB Constraints

| Layer      | Validation Type         | Example                                   |
| :--------- | :---------------------- | :---------------------------------------- |
| **DTO**    | API/business validation | `@NotBlank`, `@Email`, `@Pattern`         |
| **Entity** | DB-level constraints    | `@Column(nullable=false)`, unique indexes |

Fail fast at the API layer — never let invalid input hit the database.

---

## 🧰 Performance & Security Tips

* Use DTOs to avoid lazy-loading traps (`LazyInitializationException`).
* Don’t return entire entity graphs — shape responses explicitly.
* Always whitelist fields (DTOs) instead of blacklisting (`@JsonIgnore`).
* Encode passwords in the **service layer**, not in mappers.
* For large APIs, use **MapStruct + record DTOs** — the cleanest and fastest.

---

## 📦 Project Skeleton (DTO-friendly Layout)

```
src/
  main/java/com/example/contacts/
    entity/Contact.java
    dto/
      CreateContactRequest.java
      UpdateContactRequest.java
      ContactResponse.java
    mapper/ContactMapper.java
    repo/ContactRepository.java
    service/ContactService.java
    web/ContactController.java
```

> Optional extras: `config/SecurityConfig.java` for `PasswordEncoder`,
> `resources/application.yml` for dev DB, `test/.../ContactServiceTest.java`.

---

## 🧭 TL;DR — When to Use What

| Context                      | Approach                      |
| :--------------------------- | :---------------------------- |
| Small internal app           | Manual mapping or ModelMapper |
| Production API               | MapStruct + record DTOs       |
| Read-only queries            | Spring Data projections       |
| Bi-directional relationships | DTOs or `@JsonIdentityInfo`   |
| Security-critical data       | Always separate DTOs          |

---

**In short:**
Use **Entities** for persistence.
Use **DTOs** for communication.
Use **Mappers** as bridges.
Keep them separate — your future self will thank you.

