---
title: Q&A's 
date: 2025-11-02
tags: 
    - java
    - spring
    - validation
    - architecture
summary: A Q&A style deep dive into common questions and best practices around Jakarta Validation in Spring applications, covering method parameter validation, boundary validation strategies, and the pitfalls of database queries in validators.
aliases:
    - Questions & Answers about Jakarta Validation in Spring
---

# Questions & Answers about Jakarta Validation in Spring

## 1) “Is it only on object fields? If I put it on method parameters will it work?”

Yes, it works on method parameters (and return values), not just fields. There are **two** validation paths in Spring:

**A) MVC/Jackson path (controllers)**
When you do `@Valid @RequestBody Dto dto`, Spring MVC automatically runs bean validation on the deserialized DTO before your controller code executes.

**B) Method validation (any Spring bean: services, schedulers, Kafka listeners, etc.)**
To validate **method parameters and return values** on *any* Spring bean, add `@Validated` on the class (or method). Then put constraints directly on parameters and/or `@Valid` on complex types.

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

@Validated
@Service
public class UserService {

  // Parameter constraints work because the class is @Validated
  public User create(@Valid CreateUserCommand cmd) { … }

  public User getById(@Positive long id) { … }

  // You can validate the return value too
  public @Valid UserDto find(@NotNull String email) { … }
}
```

**How it works:** `@Validated` switches on Spring’s *method validation AOP*. Spring intercepts calls to that bean and runs Jakarta Validation on parameters/return values. Without `@Validated`, parameter annotations in services are ignored.

You can also validate programmatically anywhere:

```java
@Autowired Validator validator;

var violations = validator.validate(obj);
if (!violations.isEmpty()) { /* translate to error */ }
```

---

## 2) “Other sources like Kafka—do they use DTOs too? Same validations? Double-checking?”

Think in **boundaries**. Every external input path (HTTP, Kafka, gRPC, batch CSV, GraphQL) is a boundary. At each boundary:

* You’ll have a **message/DTO type** for that channel.
* You should **validate** it at the boundary **before** it touches your domain/services.

Examples:

**HTTP (controller)**
`@Valid @RequestBody CreateUserRequest req`

**Kafka (listener bean)**
Put `@Validated` on the listener class, then:

```java
@Validated
@Component
class UserEventsListener {

  @KafkaListener(topics = "users.create")
  public void handle(@Valid UserCreatedEvent event) {
    // safe to use 'event' here
  }

  // collections? validate elements:
  @KafkaListener(topics = "users.batch")
  public void handleBatch(@Valid List<@Valid UserCreatedEvent> events) { … }
}
```

**Schema angle:** In Kafka you often use Avro/Protobuf with a schema registry. That catches *structural* issues at deserialization time. Jakarta Validation then adds **business-shape** rules (`@NotBlank`, `@Pattern`, `@Size`, etc.). They complement each other.

**Should you reuse the same DTOs across channels?**
You *can*, but it’s usually healthier to keep **channel-specific DTOs** even if they look similar. HTTP shape and Kafka event shape tend to diverge over time (headers, metadata, batching, evolvability). Keep them separate and map into **internal commands** (the objects your service actually consumes). That keeps coupling low.

**Is this “double validation”?**
Do **not** duplicate Jakarta constraints across layers. Follow this rule:

* **Validate at the boundary** (controller, listener, batch reader) → Jakarta Validation.
* **Enforce domain invariants** inside services/domain (e.g., uniqueness, cross-aggregate rules, permission checks).
* **Let the database** have unique/foreign-key constraints as the final gate.

The checks are **different** in purpose, so it isn’t wasteful duplication—it’s defense in depth.

---

## 3) “Why do people put DB queries inside validators? Isn’t that bad practice? Shouldn’t we only validate outside data?”

You’re right to be suspicious. Folks do DB-in-validators for things like “`@UniqueEmail` must be unique in the database.” It’s tempting, but it causes problems:

* **Layer mixing:** Validators become mini-repositories → violates separation of concerns.
* **Performance:** Every validation run hits the DB (sometimes multiple times).
* **Race conditions:** You can pass the “unique” check but still lose to a concurrent insert. You needed a DB unique constraint anyway.
* **Testing brittleness:** Validators need Spring context + DB to run.
* **Context issues:** Updates need to exclude the current row (`unique except me`)—that logic gets messy inside a generic validator.

**Better patterns:**

1. **Rely on DB constraints** (unique index) as the final truth. Catch `DataIntegrityViolationException` and translate it to a clean API error.
2. **Optionally pre-check** in the service (`existsByEmail`) to fail fast for UX—but **still keep** the DB unique constraint to handle races.
3. Keep Jakarta validators **pure and stateless**. Don’t inject repositories there.

**Do we ever validate “inside” data?**

* You generally **don’t re-validate DB rows** with Jakarta annotations during normal reads; you trust your system and schema constraints.
* You **might** validate internally generated objects **before** publishing them (e.g., building an event from multiple sources) or **after transformation** in ETL flows. That’s programmatic validation for **integrity**, not user input policing.
* During **data migrations** or **external ingestion**, running validation can be useful to catch bad legacy data *before* it contaminates your domain.

---

## Quick operating checklist

* Controllers: `@Valid` on bodies/params, nested `@Valid` on fields/collections.
* Any other bean (service, listener, scheduler): put `@Validated` on the class to enable parameter/return-value validation.
* Per boundary: have a dedicated DTO/message and validate **there**.
* Services: keep **domain rules** (uniqueness, lifecycle rules, cross-entity checks). No repositories inside validators.
* Database: enforce uniqueness/foreign keys; translate violations to Problem Details.

