---
title: Monolith Project Steps
date: 2025-11-13
---

# Project Steps: Building a Modular Monolith with Spring Boot

A step-by-step guide to building a robust, maintainable backend application using vertical slice architecture with Spring Boot. This roadmap takes you from a simple monolith to a modular design, ready for future scaling and complexity.

## 0) Choose a domain with room to grow

Good options (pick one and stick to it):

* **Shopping Cart / Catalog** (recommended): Catalog, Pricing, Cart, Orders, Payments, Inventory, Delivery, Reviews.
* **Ticketing/Booking**: Events, Seats, Orders, Payments, Check-in, Refunds.
* **Learning/Content**: Courses, Lessons, Enrollment, Payments, Reviews, Search.

Why shopping-cart: it naturally yields **bounded contexts** later (Catalog, Ordering, Payments), demands **idempotency**, **consistency boundaries**, and **eventual consistency**—all the fun stuff.


---

## 1) Level 1 — Tiny, clean monolith (2–4 features)

**Goal:** ship something simple with solid edges and discipline.

**Scope (vertical slices):**

* Create/List **Categories**
* Create/List **Products** (belongs to Category)
* Add/Remove **Cart Items** (in-memory or DB)
* Checkout stub (just returns “order created” with generated id)

**Practices:**

* **Layers:** `web/`, `app/`, `domain/`, `infra/` with **feature-oriented packages** under each (e.g., `web/product`, `app/product`).
* **DTO hygiene:** request/response records under `web/<feature>/dto/...`.
* **Validation + ProblemDetail:** global exception advice, stable `problem.type` URIs via `ProblemLinks`.
* **Migrations first:** Flyway `V001__base.sql` for categories/products/carts.
* **Mapping:** MapStruct for DTO↔domain.
* **Tests:** domain unit tests, `@WebMvcTest` for controllers, `@DataJpaTest` for repos, one happy-path integration with Testcontainers.

**Deliverable shape:**

```
src/main/java/com/example/app/
  web/{category,product,cart}/...
  app/{category,product,cart}/...
  domain/{category,product,cart}/...
  infra/{jpa,mapping,config}/...
src/main/resources/db/migration/V001__base.sql
```

What you learn: thin vertical slices, stable API contracts, “migrations over magic,” and fast tests.

---

## 2) Level 2 — Make it “real API” grade

**Add:**

* **Pagination & filtering** on list endpoints.
* **Consistent JSON naming** (camel in Java, snake in JSON via Jackson config).
* **Link helpers**: `ApiLinks`, `ProblemLinks`; stop pasting strings.
* **AppProps** (typed config) + validation.
* **Observability basics**: actuator health/info, structured logging.

**Refactor moment:** normalize URI building with `UriFactory`, introduce `ProblemCatalog` (title/detail keys), tidy controllers to “edge-only” code.

What you learn: predictable API, cohesive web-edge helpers, configuration discipline.

---

## 3) Level 3 — Authentication & authorization

**Add:**

* Spring Security (JWT or Basic for demo), role model (ADMIN/USER).
* Method-level `@PreAuthorize` for admin-only endpoints (e.g., create product).
* CORS policy and CSRF posture documented.

What you learn: default-deny mindset, claim-to-role mapping, clean auth boundaries in controllers.

---

## 4) Level 4 — Orders as a real use case

**Add:**

* **Order aggregate** with invariants (snapshot prices, totals).
* **Idempotent checkout** (Idempotency-Key header stored in DB).
* **Outbox table** (but still single process) to record “OrderCreated” events.

**Refactor moment:** introduce a minimal **domain events** pattern internally (without Kafka). Keep the app a monolith; publish into an outbox table from the transaction.

What you learn: transactional boundaries, idempotency, event-first thinking without distributed pain.

---

## 5) Level 5 — Payments & external systems (still monolith)

**Add:**

* Fake **Payments** adapter (HTTP client). Simulate success/failure, timeouts, retries.
* **Saga-ish** flow inside app: reserve order → charge → confirm/cancel.
* **Circuit breaker** (Resilience4j) around payment client.

**Refactor moment:** isolate **bounded contexts** as packages/modules: `catalog`, `ordering`, `payments`. Keep them in the same codebase and database for now (a **modular monolith**).

What you learn: anti-fragile integrations, compensations, and clean adapters.

---

## 6) Level 6 — Files, search, and richer UX

**Add:**

* **Media storage** for product images (local FS first; interface for S3 later).
* **Full-text search** (Postgres `tsvector` or a tiny Elasticsearch module).
* **Price lists / discounts** (versioned, time-boxed rules).

What you learn: abstraction over infra concerns; interfaces + adapters done right.

---

## 7) Level 7 — Production-readiness pack

**Add:**

* **Tracing** (OpenTelemetry), request-id propagation.
* **Metrics**: key business counters (`orders_created_total`, `payment_failures_total`).
* **Audit trail** (who did what).
* **Blue/green-friendly config**: profiles, secrets via env/vault, no hard-coded anything.
* **Zero-downtime DB changes** (expand-migrate-contract pattern).

What you learn: the boring, important stuff that makes software live past day 3.

---

## 8) Level 8 — Hard refactors without drama

**Deliberate refactors:**

* Switch from `Long` IDs to **ULIDs/UUIDs** at API boundaries; keep DB keys stable.
* Introduce **versioned API** (`/api/v1`) and deprecate old shape gracefully.
* Extract **domain-wide value objects** (Money, SKU, Quantity) with validation.
* Move from “anemic services” to **richer aggregates** where it helps.

What you learn: controlled change, compatibility windows, and how to not break clients.

---

## 9) Level 9 — Split a context (only if it hurts)

**Criteria to split:** independent scaling/ownership, different cadence, heavy coupling to infra (e.g., Payments).

**Extract candidate:** **Payments** as a tiny service.

* Keep the rest as a modular monolith.
* Share contracts via **HTTP + ProblemDetail** and **consumer contract tests**.
* Use **outbox → message bus (Kafka/Rabbit)** for order events; payments subscribe.

What you learn: safe service extraction, contract-first, message reliability patterns.

---

## 10) Tooling & CI/CD that grows with you

* **Build:** Gradle wrapper, Spring Boot BOM, minimal starters.
* **Quality:** Spotless/Checkstyle, Error Prone or Sonar locally.
* **CI:** run unit + slice + Testcontainers; build Docker image.
* **CD:** single image promoted across `dev → test → prod` with externalized config.
* **Seed data:** a Repeatable Flyway migration for demo data (tagged for non-prod).

---

## 11) Suggested milestone checklist (copy into your vault)

**M1 — Thin Monolith**
4 slices (category, product, cart, checkout stub)
Flyway V001, DTOs, MapStruct, ProblemDetail, tests (unit/web/jpa)

**M2 — API Discipline**
Pagination, filtering, consistent JSON naming, ApiLinks/ProblemLinks, AppProps + validation

**M3 — Security**
JWT, roles, preauthorize, CORS/CSRF posture documented

**M4 — Real Orders**
Order aggregate + invariants, idempotent checkout, outbox table

**M5 — Payments**
Payment client, retries/circuit breaker, compensation path

**M6 — Infra Extras**
Media storage abstraction, search capability

**M7 — Prod-Ready**
Tracing, metrics, audit, zero-downtime migration process

**M8 — Refactors**
ULIDs at API, v1 API, shared value objects

**M9 — First Split**
Payments extracted; contract tests; outbox→bus hook-up

---

## 12) What to code next (make it tangible)

1. **V001__base.sql** with `categories`, `products`, `carts`, `cart_items` and necessary unique constraints.
2. **CreateCategory** vertical slice end-to-end, including `409 DuplicateCategory` ProblemDetail with stable `type`.
3. **ListProducts** with pagination + filtering (`?categoryId=&q=`).
4. **AddToCart** with quantity rules and idempotency on repeated POSTs.

From there, follow the levels. Keep slices small, refactor steadily, and write one integration test per feature to keep your courage high.

