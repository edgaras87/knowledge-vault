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
    - API Input and Command Structuring in Java Spring
---

# **API Inputs, Commands, DTOs & Boundaries — Full Concept Doctrine**

### **Why this exists**

When building APIs, it’s easy to blend HTTP types, domain logic, and application data into one soup. It works until:

* You add second transport (Kafka/CLI/batch)
* You need JSON quirks
* You need OpenAPI annotations
* You start worrying about backwards compatibility
* You introduce idempotency and domain events
* You're aiming for realistic professional architecture

This doctrine defines **how to structure request types now** so you evolve smoothly later.

---

## **Fundamental Idea**

You don’t start by over-architecting.
You start **simple, clean, transport-agnostic**.

But you place one tiny fence:

> **No Jackson or OpenAPI annotations outside the `web` layer.**

From that rule, everything else flows naturally.

---

## **Key Type Categories**

### **1. Web Request DTO**

Represents exactly what comes over HTTP.

Lives only in `…web`.

May contain:

* `@JsonProperty`, `@JsonFormat`, etc.
* OpenAPI `@Schema` etc.
* Multipart types (`MultipartFile`)
* Only shapes the wire, not your logic

Example:

```java
public record CreateUserRequest(
  @JsonProperty("user_name")
  String name,
  @Schema(example="a@b.com")
  String email
) {}
```

### **2. Application Input / Command**

What your business logic actually consumes.
Transport-agnostic, clean, validated, normalized.

Example:

```java
public record CreateUserInput(
  @NotBlank String name,
  @Email @NotBlank String email
) {
  public CreateUserInput {
    if (name != null) name = name.trim();
    if (email != null) email = email.trim().toLowerCase();
  }
}
```

### **3. Domain Entity**

Persistence model (JPA etc). Never leave the domain layer.

### **4. Response DTO**

What your API sends back. Lives in `…web`.

---

## **Single-Type Phase (Early Stage / Portfolio / Simple App)**

Bind the input directly in controller:

```java
@PostMapping
ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserInput in) {
  var created = service.create(in);
  return ResponseEntity.created(...).body(mapper.toResponse(created));
}
```

No web DTO yet. This works **until** something forces separation.

---

## **The Tripwires — When to Split to `Request → Input`**

You split the moment you hit any one of these:

1. Need a JSON rename / custom date format
   (`@JsonProperty`, `@JsonFormat`, polymorphic JSON)

2. Need OpenAPI field annotations
   (examples, descriptions, constraints)

3. Add another input channel
   (Kafka listener, CLI tool, batch import, gRPC, GraphQL)

4. Handling multipart uploads
   (`MultipartFile` can't exist in app layer)

5. Public API stability matters
   (must evolve server without breaking clients)

---

## **What Happens When a Tripwire Fires**

You **do NOT rewrite your system**.

You **only**:

* Add a `CreateUserRequest` type to `…web`
* Map to existing `CreateUserInput`
* Leave service untouched

Example migration:

Before:

```
Controller -> CreateUserInput -> Service
```

After:

```
Controller -> CreateUserRequest -> CreateUserInput -> Service
```

The rest of the project remains unchanged.

---

## **Mixing Phases is Allowed**

Yes, you can have some endpoints still using `*Input` directly and others already using `*Request → *Input`.

Consistency matters eventually — not immediately.

This is **gradual evolution**, not religion.

---

## **Validation Position Choices**

### Option A: Validate at the Web Edge

Constraints on `CreateUserRequest`.

Pros: invalid JSON dies early
Cons: other transports repeat rules

### Option B: Validate On the Input (Recommended for multiple channels)

Constraints on `CreateUserInput`.

Pros: single validation point for REST, Kafka, CLI
Cons: controller triggers validation differently

Either way is fine — just choose consciously.

---

## **Mapping Philosophy**

### Always one-way:

* Request → Input
* Input → Entity (create)
* Input → @MappingTarget Entity (update)
* Entity → Response

### Never round-trip DTO ↔ Entity or DTO ↔ Input both ways.

One-way flows prevent spaghetti.

---

## **Entity Rules**

Never expose entity outside the domain layer.
Never accept entity in a controller.
Never serialize entity to JSON.

Entities are not API contracts.

---

## **Folder Structure**

```
users/
  web/
    UserController.java
    CreateUserRequest.java (if needed)
    UserResponse.java
  app/
    CreateUserInput.java
    UpdateUserInput.java
    UserService.java
  domain/
    User.java
    UserRepository.java
```

---

## **Input vs Command Naming**

### **Input**

* Data passed into a use-case
* Simple CRUD
* Sync processing
* No eventing/idempotency
* Where you start

### **Command**

* Intentful business action (“do this”)
* Write side in CQRS
* May be async
* May require idempotency
* May emit events
* Where you evolve

**Inputs are the beginner-friendly version; Commands are the “growing up” version.**

---

## **How To Evolve Input → Command**

Rename `CreateUserInput` → `RegisterUserCommand`

Add capabilities:

* `idempotencyKey`
* `initiatedBy`
* event publishing after success
* handler instead of simple service call

No architectural shock if you followed rules.

---

## **Realistic Evolution Timeline**

| Stage                  | Style                      | Why                         |
| ---------------------- | -------------------------- | --------------------------- |
| Learning CRUD          | `*Input` only              | no ceremony                 |
| Portfolio project      | `*Input` + Response DTO    | clean edges, fast progress  |
| First JSON quirk       | add `*Request`             | separation begins           |
| Multiple transports    | `*Request → *Input` always | consistent boundary         |
| Scaling business logic | rename to `*Command`       | semantics matter            |
| Advanced system        | command bus, events        | now you’re in pro territory |

---

## **Mental Models**

* “**Input** feeds a function.”
* “**Command** triggers a business act.”
* “**DTO** describes the wire.”
* “**Boundary** protects the core.”
* “Evolve in place — don’t overbuild early.”

---

## **Q&A**

**Q: When do I start with DTO + Input?**
Only when a tripwire hits.

**Q: Can I leave some endpoints using `Input` and others using `Request → Input`?**
Yes — migrate gradually.

**Q: Does adding Kafka force a DTO?**
Not automatically — but it's a hint you’re entering multi-adapter territory. Best practice is one canonical `*Input/*Command` + mapping per adapter.

**Q: If later I need `@JsonProperty`, do I change Input?**
No. Add `*Request` and map. Input stays clean.

**Q: Is Input wrong for real production?**
No. Many real systems use Inputs until CQRS needs Commands.

**Q: When do I switch to Command?**
When you need idempotency, events, async, or multi-write path semantics.

**Q: Will my services break when I split?**
No. Only controllers change. Services keep same `Input`.

---

## Final mantra

> **Start simple.
> Guard your boundaries.
> Split only when reality demands.
> Rename Inputs to Commands when business complexity appears.**

You avoid premature complexity **and** avoid cornering future-you into rewrites.

