---
title: Ports & Adapters 
date: 2025-11-04
tags: 
    - spring
    - architecture
    - hexagonal
    - ports-and-adapters
    - clean-architecture
summary: A guide to organizing Spring Boot applications using the Ports-and-Adapters (Hexagonal) architecture for clean, maintainable, and testable codebases.
aliases:
    - Ports-and-Adapters Architecture in Spring Boot  
---

# Ports-and-Adapters Architecture in Spring Boot

When building a Spring Boot application, organizing your codebase using the Ports-and-Adapters (Hexagonal) architecture can lead to a clean, maintainable, and testable structure. This approach separates the core business logic from external concerns like web frameworks and databases.

### Core Concepts

Here’s a **complete, production-shaped** layout using your edges→core rhythm, with the “service” implementations visible under `app/.../service`. It’s lean, extensible, and won’t collapse when you add more features.

```text
shopping-cart/
├─ build.gradle / pom.xml
├─ README.md
├─ .editorconfig
├─ .gitignore
├─ docker-compose.yml                      # local MySQL/Postgres, etc.
└─ src
   ├─ main
   │  ├─ java/com/example/app/
   │  │  ├─ AppApplication.java            # Spring Boot entrypoint
   │  │  ├─ config/
   │  │  │  ├─ WebConfig.java              # CORS, Pageable, message converters
   │  │  │  ├─ JacksonConfig.java          # ObjectMapper: dates, enums, modules
   │  │  │  └─ ValidationConfig.java       # Method validation, i18n
   │  │  ├─ docs/
   │  │  │  └─ OpenApiConfig.java          # springdoc / Swagger UI
   │  │  ├─ error/
   │  │  │  ├─ ProblemTypes.java           # catalog of RFC7807 type URIs
   │  │  │  └─ GlobalExceptionHandler.java # @RestControllerAdvice → ProblemDetail
   │  │  ├─ security/
   │  │  │  ├─ SecurityConfig.java         # stateless security (JWT/demo)
   │  │  │  └─ DemoAuthFilter.java         # optional: "Bearer demo"
   │  │  ├─ web/
   │  │  │  └─ user/
   │  │  │     ├─ UserController.java
   │  │  │     ├─ dto/
   │  │  │     │  ├─ request/
   │  │  │     │  │  ├─ CreateUserRequest.java
   │  │  │     │  │  ├─ UpdateUserRequest.java
   │  │  │     │  │  └─ PatchUserRequest.java
   │  │  │     │  └─ response/
   │  │  │     │     ├─ UserListItemResponse.java
   │  │  │     │     ├─ UserDetailResponse.java
   │  │  │     │     ├─ UserSummaryResponse.java
   │  │  │     │     └─ envelopes/
   │  │  │     │        ├─ SingleEnvelope.java
   │  │  │     │        ├─ CollectionEnvelope.java
   │  │  │     │        └─ PageEnvelope.java
   │  │  │     └─ mapper/
   │  │  │        ├─ UserInputMapper.java   # Request → Command (MapStruct)
   │  │  │        └─ UserOutputMapper.java  # Entity/Projection → Response
   │  │  ├─ app/
   │  │  │  └─ user/
   │  │  │     ├─ usecase/                  # contracts (ports) for web layer
   │  │  │     │  ├─ CreateUserUseCase.java
   │  │  │     │  ├─ UpdateUserUseCase.java
   │  │  │     │  ├─ PatchUserUseCase.java
   │  │  │     │  └─ SearchUsersUseCase.java
   │  │  │     ├─ service/
   │  │  │     │  └─ DefaultUserService.java   # implements *UseCase interfaces
   │  │  │     └─ command/                  # input models for app layer
   │  │  │        ├─ CreateUserCommand.java
   │  │  │        ├─ UpdateUserCommand.java
   │  │  │        ├─ PatchUserCommand.java
   │  │  │        └─ UserSearchCommand.java
   │  │  ├─ domain/
   │  │  │  └─ user/
   │  │  │     ├─ User.java                 # aggregate (business rules here)
   │  │  │     ├─ UserId.java               # value object (optional)
   │  │  │     ├─ UserRepository.java       # domain port
   │  │  │     └─ UserSpec.java             # optional: search/filter spec
   │  │  └─ infra/
   │  │     └─ user/
   │  │        ├─ JpaUserEntity.java
   │  │        ├─ SpringDataUserRepository.java   # implements domain port
   │  │        └─ UserJpaMapper.java              # only if JPA ≠ domain
   │  └─ resources/
   │     ├─ application.yml                 # profiles, DB, logging levels, CORS
   │     ├─ application-test.yml
   │     ├─ db/migration/
   │     │  └─ V1__init.sql                 # Flyway baseline
   │     ├─ logback-spring.xml              # sane, structured logs
   │     └─ static/                         # optional
   └─ test
      ├─ java/com/example/app/
      │  ├─ web/user/UserControllerTest.java    # @WebMvcTest / @SpringBootTest
      │  ├─ app/user/DefaultUserServiceTest.java# unit tests for service
      │  ├─ domain/user/UserTest.java           # entity invariants
      │  └─ infra/user/UserRepositoryIT.java    # @DataJpaTest + Testcontainers
      └─ resources/
         └─ application-test.yml
```

### Wiring rules (so it stays clean)

* **Controllers** depend on `*UseCase` **interfaces** only.
* **DefaultUserService** implements multiple `*UseCase`s; it depends on **domain ports** (`UserRepository`) and invokes domain logic on `User`.
* **Infra adapters** implement domain ports. No controller or service should depend on JPA classes.
* **Transactions** live in the **service** (`@Transactional` at class/method level).
* **Mapping**: edge maps in `web/.../mapper`, infra map only if your JPA entity differs from the domain aggregate.
* **Errors**: throw domain/app exceptions; translate once in `GlobalExceptionHandler` to **ProblemDetail** using `ProblemTypes`.






## **DefaultUserService** implements multiple `*UseCase`s; 

!!! note
    **DefaultUserService** implements multiple `*UseCase`s; it depends on **domain ports** (`UserRepository`) and invokes domain logic on `User`.

In the “ports & adapters” style, your **application service** (e.g., `DefaultUserService`) orchestrates a use case by:

1. taking an input (`*Command`),
2. calling **domain logic** (methods on your aggregates or domain services), and
3. persisting via a **domain port** (`UserRepository` interface that lives in `domain`).

That line—“depends on domain ports and invokes domain logic”—means two very specific dependency choices:

1. **Outward dependency:** `DefaultUserService` knows only the interface `UserRepository` (a *port* declared in `domain`), not JPA/Spring Data classes.
2. **Inward delegation:** business rules are on the `User` aggregate (or a domain service), not inside the application service.

### Concrete mini-illustration

**Domain (core)**

```java
// domain/user/UserRepository.java  (PORT)
public interface UserRepository {
  void save(User user);
  Optional<User> findByEmail(Email email);
  Optional<User> findById(UserId id);
  Page<User> search(UserSpec spec, Pageable pageable);
}

// domain/user/User.java  (AGGREGATE ROOT)
public class User {
  private final UserId id;
  private Email email;
  private String name;

  private User(UserId id, Email email, String name) { /* invariant checks */ }

  public static User create(Email email, String name) {
    // enforce domain rules: email format, name policy, uniqueness checked in app layer
    return new User(UserId.newId(), email, name);
  }

  public void rename(String newName) {
    // invariant: name not blank, no leading/trailing spaces, maybe length limit
    this.name = newName.trim();
  }

  public void changeEmail(Email newEmail) {
    // policy: maybe forbid same domain? record audit? etc.
    this.email = newEmail;
  }

  // getters...
}
```

**Application (orchestrator)**

```java
// app/user/usecase/CreateUserUseCase.java
public interface CreateUserUseCase {
  UserId handle(CreateUserCommand cmd);
}

// app/user/service/DefaultUserService.java
@Service
@Transactional
@RequiredArgsConstructor
class DefaultUserService implements
    CreateUserUseCase, UpdateUserUseCase, PatchUserUseCase, SearchUsersUseCase {

  private final UserRepository users; // ← depends on domain PORT, not JPA class

  @Override
  public UserId handle(CreateUserCommand cmd) {
    // app-level policy: check email uniqueness by querying the port
    users.findByEmail(new Email(cmd.email()))
         .ifPresent(u -> { throw new EmailAlreadyUsed(cmd.email()); });

    // invoke DOMAIN logic (factory enforces invariants)
    User user = User.create(new Email(cmd.email()), cmd.name());

    // persist via PORT
    users.save(user);
    return user.getId();
  }

  @Override
  public void handle(UpdateUserCommand cmd) {
    User user = users.findById(cmd.id())
                     .orElseThrow(() -> new UserNotFound(cmd.id()));
    // domain mutation
    if (cmd.name() != null) user.rename(cmd.name());
    if (cmd.email() != null) user.changeEmail(new Email(cmd.email()));
    // flush-on-commit; no direct JPA here
  }

  // ...other use cases...
}
```

**Infrastructure (adapter implements the port)**

```java
// infra/user/SpringDataUserRepository.java
@Repository
@RequiredArgsConstructor
class SpringDataUserRepository implements UserRepository {
  private final JpaUserDao dao;           // Spring Data interface
  private final UserJpaMapper mapper;     // maps between domain <-> JPA entity

  @Override
  public void save(User user) {
    dao.save(mapper.toJpa(user));
  }

  @Override
  public Optional<User> findByEmail(Email email) {
    return dao.findByEmail(email.value()).map(mapper::toDomain);
  }

  // etc.
}
```

The **dependencies all point inward**:

* Controller → `CreateUserUseCase` (interface)
* `DefaultUserService` → `UserRepository` (domain port) and `User` (domain model)
* Infra adapter → implements `UserRepository` (depends outward on Spring Data, inward on domain types)

---

## Are there other ways that *don’t* depend on domain ports?

Yes. They trade clarity for speed, and that’s sometimes fine—be deliberate.

### 1) Controller talks directly to Spring Data repositories (no app service)

* **What it looks like:** `UserController` injects `JpaRepository<UserEntity, Long>` and calls it.
* **Pros:** fastest to code for CRUD prototypes.
* **Cons:** business rules end up in controllers or entities/JPA callbacks; hard to test; coupling to Spring Data leaks everywhere; refactors hurt when logic grows.

When acceptable: tiny CRUD apps, admin back-office, or short-lived prototypes.

---

### 2) Application service depends directly on Spring Data (no domain port)

* **What it looks like:** `DefaultUserService` injects `UserJpaRepository` (Spring Data interface).
* **Pros:** fewer files; less mapping boilerplate.
* **Cons:** application layer now depends on infrastructure details; swapping persistence (JPA → JDBC, Mongo) becomes invasive; unit tests need Spring/DB or heavy mocking.

When acceptable: simple domains where the datastore *is* the model, or when you know you won’t change persistence.

---

### 3) One handler per use case (CQRS-ish)

* **What it looks like:** `CreateUserHandler implements CreateUserUseCase`, `UpdateUserHandler implements UpdateUserUseCase`, etc. No single “DefaultUserService.”
* **Pros:** clean isolation per use case; easy to scale across teams; nice with pipelines (logging, tracing) around handlers; plays well with mediators (e.g., MediatR-style).
* **Cons:** more classes; can feel fragmented for small projects.

When to use: big codebases, complex workflows, or heavy read/write separation.

---

### 4) Anemic domain model (business rules in service)

* **What it looks like:** `User` is a data bag; all rules live in application service.
* **Pros:** simple entities; logic easy to find in services.
* **Cons:** domain invariants not protected where the data lives; easier to bypass rules; harder to compose behaviors.

When acceptable: data-centric apps with light rules, strong validation at boundaries, and low risk of invariants being violated elsewhere.

---

### 5) Rich domain + domain services (when logic spans aggregates)

* **What it looks like:** `UserDomainService` encapsulates rules that involve multiple aggregates or policies; app service orchestrates and calls domain services.
* **Pros:** keeps invariants in the domain, avoids bloated entities, clarifies cross-aggregate rules.
* **Cons:** extra abstraction; overkill for simple CRUD.

When to use: genuinely complex rules (billing, scheduling, allocation).

---

## Quick decision guide

* **Need flexibility, testability, and clear boundaries?**
  Use **ports** in `domain`, an **app service** that depends on those ports, and **rich aggregates** for rules.

* **Optimizing for speed on a tiny service?**
  Let controller or app service depend directly on Spring Data. Revisit later if it grows fangs.

* **Team is large / workflows complex?**
  **One handler per use case** often wins.

* **Rules are heavy and subtle?**
  Keep them on the **aggregate** or a **domain service**; don’t let controllers or JPA entities carry that weight.

The version you’re building (app service depends on domain ports; invokes behavior on aggregates) is the “boringly correct” baseline. It keeps options open, scales with you, and makes tests cheap. Next natural step is to add one more feature (e.g., “deactivate user”) that touches multiple rules—so you can feel the payoff of having the logic in the domain rather than in the controller.
