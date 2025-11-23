---
title: End-to-End Steps
date: 2025-11-13
---



# Spring in the real world: end-to-end steps

Here’s what “Spring in the real world” usually looks like, end-to-end.

## 1) Start with the domain, but stay pragmatic

* Agree on **ubiquitous language** with product/UX (“category”, “sku”, “cart item”, “checkout”).
* Identify **bounded contexts** if the domain is large (e.g., Catalog, Orders, Payments). Don’t force microservices early; a modular monolith is fine.
* Capture a small backlog of **thin vertical slices** (a user-visible outcome that cuts through API → app/service → domain → persistence). Example: “Create category” (with name uniqueness) is a slice.

## 2) Turn slices into concrete work

For each slice:

* A tiny **story**: “As an admin, I can create a category with a unique name.”
* **Acceptance criteria**: unique name, returns 201 + Location header, proper error if duplicate (ProblemDetail).
* **Non-functionals** you’ll respect from day one: validation, logging, metrics, basic tests.

## 3) Architecture shape most teams use

* **Hexagonal / layered hybrid** is common:

  * **web**: controllers, request/response DTOs, exception translators, link/URI helpers.
  * **app** (use cases): application services orchestrating domain logic.
  * **domain**: entities/aggregates, value objects, domain services, policies.
  * **infra**: repositories (JPA), mapping (MapStruct), messaging adapters, config.
* Keep packages **feature-oriented** under each layer (e.g., `web/category`, `app/category`, etc.) so the code scales beyond 5 entities.

## 4) Data first: migrations over magic

* Design the **minimum schema** for the slice.
* Write a **Flyway** migration (repeatable and versioned).
* Map with **JPA**; keep entities clean (business rules) and shield them from the web with DTOs.

## 5) API design that doesn’t haunt you later

* Name resources plainly: `/categories`.
* Stable **identifiers** (usually numeric or ULIDs). No leaking internal DB details.
* **ProblemDetail (RFC 7807)** for errors + predictable **problem types** (links).
* Add **link helpers** (e.g., `ProblemLinks`, `ApiLinks`) to avoid URI string soup.
* Decide **naming**: Java uses camelCase; JSON can be snake_case via Jackson config; keep it consistent and documented.

## 6) Implementation rhythm (per slice)

1. **Contract**: endpoint, request/response examples, error cases.
2. **Schema**: Flyway VXXX__create_category.sql.
3. **Domain**: `Category` aggregate + invariants (e.g., normalized name).
4. **Repository**: Spring Data JPA (unique constraint enforced at DB + checked in service).
5. **App service (use case)**: `CreateCategory` orchestrates repo calls + policies.
6. **Web**: controller + request/response DTOs; global exception advice → ProblemDetail.
7. **Mapping**: MapStruct between DTOs ↔ domain models where it reduces boilerplate.
8. **Tests**: unit (domain), slice (`@DataJpaTest`, `@WebMvcTest`), a happy-path integration test (Testcontainers if a real DB).
9. **Observability**: log key events, increment a metric (e.g., `category_created_total`).

## 7) Testing pyramid that actually runs fast

* **Domain unit tests**: zero Spring, milliseconds.
* **Slice tests**:

  * `@WebMvcTest` for controller/JSON/validation.
  * `@DataJpaTest` for repository/query behavior.
* **Integration**: minimal but real—boot the app, hit the endpoint against a Testcontainers DB.
* **Contract tests** if you have multiple teams/clients.

## 8) Quality guardrails

* **Static analysis**: Checkstyle/Spotless, Error Prone or Sonar.
* **Dependency management**: pin versions (Spring Boot BOM), keep starters minimal.
* **Build reproducibility**: containerized builds or Gradle wrapper.

## 9) Delivery flow

* **Branching**: small PRs per slice, peer review.
* **CI**: run unit + slice tests, build artifact, publish image.
* **CD**: promote the same artifact across **dev → test → prod** with externalized config (profiles + env vars).
* **Feature flags** for risky features (toggle via properties).

## 10) Configuration & environments

* Centralize `AppProps` with typed config (records), validate with Bean Validation.
* Keep secrets out of git; inject via env or vault.
* Provide a **/actuator/health** and basic **info** for runtime checks.

## 11) Security from day one (even if simple)

* Spring Security on, default-deny.
* Start with simple auth (Basic/JWT) and role checks at the controller.
* Add CSRF/CORS as needed; document it.

## 12) Observability & readiness

* Structured **logging** (JSON in containers).
* **Micrometer** metrics (HTTP, DB, JVM).
* Tracing (OpenTelemetry) when you add more services.

## 13) DDD—where teams draw the line

* Use **DDD language and boundaries** to stay aligned, not to cosplay academia.
* Keep aggregates small; enforce invariants close to the data.
* Introduce **domain events** only when you actually need cross-component reactions or outbox patterns.
* Prefer **modular monolith** early; split into services when pain demands it (deployment cadence, independent scaling, team ownership).

## A tiny example slice: “Create Category”

* Contract: `POST /categories` → 201 + `Location: /categories/{id}`, duplicate → `409` with `type: /problems/duplicate-category`.
* DB: unique index on `categories(normalized_name)`.
* Domain: `Category` ensures `name` is trimmed/non-blank; normalize once.
* App: `CreateCategoryHandler` checks via repo, saves, returns id.
* Web: `CreateCategoryRequest(name)`, `CategoryResponse(id,name)`.
* Errors: `DuplicateCategoryProblem` via global advice → ProblemDetail.
* Tests: domain unit test for normalization/blank; repo uniqueness; controller 201 + 409; one integration test validates the whole path.

That’s the real rhythm: learn the domain, slice thin, keep edges clean, enforce rules where they matter, and ship iteratively with tests and observability. 

