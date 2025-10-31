---
title: Core Flow
date: 2025-10-31
tags: 
    - spring
    - architecture
    - best-practices
    - software-architecture
    - dto
summary: Your definitive map for building Spring Boot applications that never rot. Clean data in, truth enforced, invariants protected.
aliases:
    - Spring Boot Core Flow Summary
---

# 🌱 Spring Boot Core Flow — Summary

Spring apps breathe in **HTTP requests** and breathe out **domain truth**. The air passes through a clear sequence:

`Controller → DTO → Service → Entity/Repo → Response DTO`

Each layer has one job. When they stay in their lane, systems stay clean and your brain stays sane.

### Controllers

The bouncers of the API club.
They accept requests, attach `@Valid` DTOs, and hand off to services. They **never** do business logic.

```java
@PostMapping
public ResponseEntity<ProductResponseDto> create(@Valid @RequestBody ProductCreateDto dto) {
    return ResponseEntity.ok(service.create(dto));
}
```

---

### DTOs

The suit-and-tie data ambassadors.
They **sanitize and validate** user input and define the shape of responses.
Entities should never be exposed — that's like giving your DB keys to strangers.

```java
public record ProductCreateDto(
 @NotBlank @Size(max=64) String sku,
 @NotNull @Positive Long priceCents,
 ...
) {}
```

**DTO rules**:
• Input = strict, limited fields
• Output = only what the client should see
• Validation lives here, not in Entities

---

### Services

The business logic dojo.
This is the brain of your application — rules, permissions, workflows.

```java
@Service
public class ProductService {
  public ProductResponseDto create(ProductCreateDto dto) {
    if (repo.existsBySku(dto.sku())) throw new IllegalArgumentException("Duplicate SKU");
    var saved = repo.save(ProductMapper.toEntity(dto));
    return ProductMapper.toResponse(saved);
  }
}
```

**Service rules**:
• one method = one business action
• enforce invariants (unique SKU, stock >= 0, etc.)
• transactional where needed

---

### Entities & Repository

Entities are **database truth** objects — nothing more.
Repositories are boring on purpose. You want boring persistence.

```java
@Entity
public class Product {
 @Id @GeneratedValue private Long id;
 @Column(nullable=false, unique=true) String sku;
 Instant createdAt;
 @PrePersist void init(){ ... }
}
```

`JpaRepository` gives CRUD, plus custom finders (`findBySku()`).

**Entity rules**:
• DB constraints: nullability, lengths, indexes
• no request validation annotations
• lifecycle hooks allowed
• avoid polluting with presentation logic

---

### Mappers

Little translators. Bridge DTO ↔ Entity.

Manual one is fine at first. Later: **MapStruct**.

---

### Config (`application.yml`)

Set your datasource, JPA mode, and debug flags.
Don’t over-tune until needed.

---

### Tests

Just one test already forces correctness discipline.

```java
@SpringBootTest
class ProductServiceTest {
 @Test void create_ok() { ... }
}
```

Tests teach you to think in behavior, not hope.

---

### Working rhythm

You’re building **muscle memory**, just like learning kettlebells or scales on a guitar:

1. Controller → takes DTO
2. Validate input
3. Service → enforces business rules
4. Entity + Repo → persistence
5. Return response DTO
6. Config + test + run

Do that loop a hundred times and Spring stops feeling “enterprise wizardry” and starts feeling like breathing.

---

### Spirit of the architecture

The deeper theme here: **boundaries protect sanity**.

* DTOs protect the DB from the internet
* Services protect rules from controllers
* Entities protect storage from API churn
* Tests protect future-you from present-you

Loose boundaries mean pain. Clear boundaries mean teams scale and correctness becomes habit.

In the future you’ll sprinkle spices:
MapStruct, TDD, Projections, ProblemDetail, Flyway, distributed tracing… but this flow is your spine.

