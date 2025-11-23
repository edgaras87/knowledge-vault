---
title: Spring Domain — Future Blueprint 
date: 2025-11-13
---

# Domain Feature Blueprint — Future Additions

**Path:** `docs/cheatsheets/domain/blueprint.md`
**Context:** `web / app / domain / infra` (Java 17+, MapStruct, JPA, Flyway)

---

## 1) Core you already have (baseline)

* **Aggregate root**: behavior + invariants (e.g., `Category`)
* **Value Objects**: `Slug`, `Money`, `Quantity`, `Email` (as needed)
* **Typed ID**: `CategoryId`
* **Repository (port) interface**
* **Creation / Rehydration** factories
* **Domain exceptions** (e.g., `DuplicateCategoryException`)
* **`createdAt` policy** (DB default *or* app clock)

> Baseline DoD: domain compiles with **no** Spring/JPA/Jackson imports; unit tests cover rules; repo interface expresses the use-cases your app actually needs.

---

## 2) Next-level domain building blocks (add when needed)

### 2.1 Domain Events (internal)

Use when other parts of the domain must react to a change.

```
domain/category/
  events/
    CategoryCreated.java         // record with CategoryId, Slug, occurredAt
```

* Fire **inside** domain methods (e.g., after `createNew`).
* App layer subscribes to orchestrate side effects.
* Keep them **in-process**; for cross-service, see Integration Events (8.3).

### 2.2 Policies & Domain Services

Pure business rules that don’t sit neatly in one aggregate.

```
domain/pricing/DiscountPolicy.java
domain/catalog/NamingPolicy.java
```

* Stateless, framework-free, easy to test.

### 2.3 Specifications (query intentions, not SQL)

Model **business filters** without committing to DB details.

```
domain/product/spec/ProductByCategory.java
domain/product/spec/ProductSearch.java
```

* Repos accept specs (`findAll(ProductSpec, Pageable)`) and infra translates them.

### 2.4 Soft Delete & Retention

If you need “undo” or compliance.

* VO: `DeletedAt` (nullable) or explicit state `Active/Archived/Deleted`.
* Rule: domain forbids operations on deleted.
* DB: partial index on active rows; archiver job in app/infra.

### 2.5 Versioning & Concurrency

When concurrent updates matter.

* Domain field: `version` (long) with **semantic** meaning (optional).
* Infra: JPA `@Version` for optimistic locking.
* Exception: map to `409` with proper ProblemDetail type.

### 2.6 Localization (optional)

If labels are multilingual.

* VO: `LocalizedText` or `NameTranslations` (`Map<Locale, String>`).
* Rule: default locale must exist; limit keys.
* Infra: store as JSONB (Postgres) or normalized table.

### 2.7 Tenancy (optional)

* VO: `TenantId` (typed).
* Rule: every aggregate scoped by tenant; repos automatically constrain.
* Test: ensure cross-tenant leakage is impossible.

---

## 3) Feature package template (copy this)

```
src/main/java/com/example/shop/domain/<feature>/
  <Feature>.java                     # aggregate root
  <Feature>Id.java                   # typed id
  vo/
    <Feature>Code.java               # optional, like Slug/Handle
    ... (other VOs)
  events/
    <Feature>Created.java            # optional
  spec/
    <Feature>BySomething.java        # optional
  <Feature>Repository.java           # port
  exceptions/
    <Feature>RuleViolation.java
```

**Example – orders:**

```
domain/order/
  Order.java
  OrderId.java
  vo/
    OrderNumber.java
    Money.java
  events/
    OrderPlaced.java
    OrderPaid.java
  spec/
    OrdersByCustomer.java
  OrderRepository.java
  exceptions/
    OrderAlreadyPaid.java
```

---

## 4) Aggregate lifecycle patterns (decide early)

* **Creation**: `createNew(input, now)` derives VOs, enforces rules.
* **Mutation**: methods like `rename`, `addLineItem`, `applyDiscount` return a **new** instance or mutate private state behind invariants (your call; keep rules centralized).
* **Rehydration**: `rehydrate(id, canonicalFields..., timestamps...)` **never derives**.
* **Deactivation/Deletion**: explicit methods; favor soft delete if audit matters.
* **Event emission**: aggregate returns a list of `DomainEvent` produced during the call (app layer publishes).

---

## 5) Cross-cutting value objects (ready to reuse)

* **Money**: `record Money(BigDecimal amount, Currency currency)`

  * factory `of(String amount, String currency)`; arithmetic inside; scale rules.
* **Percentage**: `record Percentage(int value)`; 0–100 guard.
* **Email**: `record Email(String canonical)`; `ofUserInput` vs `ofCanonical`.
* **TimeWindow**: `record TimeWindow(Instant start, Instant end)`; guards ordering.
* **Handle**: like `Slug` but user-assignable; reserved words, min/max length.
* **Name**: `record Name(String value)`; optional trimming & forbidden chars policy.

> Put truly shared VOs in `domain/common/`; keep feature-specific VOs under the feature.

---

## 6) Repository contract blueprint

```java
public interface <Feature>Repository {
  <Feature> save(<Feature> agg);                     // returns rehydrated
  Optional<<Feature>> findById(<Feature>Id id);
  Page<<Feature>> findAll(Pageable pageable);

  // Intent-based lookups (don’t leak SQL):
  Optional<<Feature>> findByCode(<Feature>Code code); // e.g., Slug/Handle
  boolean existsByCode(<Feature>Code code);

  // Optional: specs
  Page<<Feature>> findAll(<Feature>Spec spec, Pageable pageable);
}
```

**Rules**

* Use **typed parameters** everywhere.
* Return domain types, never entities.
* Keep pagination at repo boundary; sorting fields are **domain** fields.

---

## 7) Error & problem catalog (bridge to web)

* Domain exceptions are **specific** (`CannotMergeCartAcrossTenants`, `DuplicateCategory`).
* App layer maps them to a **stable ProblemDetail type URI**, e.g. `/problems/duplicate-category`.
* Keep a small **ProblemKey enum** in web/support that maps to i18n codes.

---

## 8) Evolution paths (when the app grows fangs)

### 8.1 CQRS Light (per feature)

* Domain remains the write model.
* Add read models/projections (in `infra/readmodel/...`) for list/search screens.
* App layer uses read model for queries, domain for commands.

### 8.2 Outbox / Integration Events

* Domain raises internal events.
* App/infra translate some to **integration events** (schema-versioned) and store in **outbox** table.
* A reliable publisher ships them to message broker.

### 8.3 Anti-corruption layer (external systems)

* Create VOs that protect your domain from external formats.
* Translate at the edge; domain never sees third-party quirks.

---

## 9) Testing blueprint (repeatable per feature)

**Unit (domain only)**

* Constructors/factories guard invariants.
* Mutators enforce rules and produce expected `DomainEvent`s.

**Contract tests (repo interface)**

* Same test suite for any repo impl (JPA/Testcontainers, in-memory fake).

**App/service tests**

* Orchestrate domain calls + repo interactions; verify error mapping.

**Fixture style**

```
DomainFixtures.category("Books")      // returns Category.new(...)
MoneyFixtures.eur("19.99")
```

---

## 10) Checklists you’ll reuse

### 10.1 New feature checklist

* Define aggregate root + typed ID.
* List invariants in comments/tests first.
* Identify VOs (codes, money, email, quantities).
* Decide slug/handle policy (immutable? derived?).
* Define repo interface (lookups you actually need).
* Define domain exceptions.
* Write unit tests for create/mutate/rehydrate.
* Add Flyway migration (tables + unique constraints on canonical columns).
* Implement JPA entity + MapStruct mapper + repo adapter.
* App handler + web DTOs + ProblemDetail mapping.

### 10.2 Rename policy (if applicable)

* Slug immutable?

  * Yes: only `name` changes; add alias table for redirects if needed.
  * No: re-derive slug; maintain redirects.

### 10.3 Time policy

* Source of truth (`DB now()` vs `TimeFacade`).
* Always UTC; never `LocalDateTime` without zone context.

---

## 11) Example file tree (feature “product”)

```
domain/product/
  Product.java
  ProductId.java
  vo/
    ProductSlug.java
    Money.java
  events/
    ProductCreated.java
  spec/
    ProductByCategory.java
  ProductRepository.java
  exceptions/
    DuplicateProduct.java
```

---

## 12) Pitfalls to dodge (pin on your wall)

* Recomputing derived fields on **read** (causes historic drift).
* Leaking JPA annotations into domain.
* Using Lombok builders on aggregates (bypasses invariants).
* Putting generic helpers in `common/` too early (wait until 2+ features share them).
* Repo methods that mirror tables instead of **use-cases**.

---

## 13) “Is this enough?” (yes—then iterate)

This blueprint gives you a **stable center**:

* Entities + VOs that **mean** something,
* Repos that express **intent**,
* Clear paths for events, specs, tenancy, localization, and CQRS when you need them.


