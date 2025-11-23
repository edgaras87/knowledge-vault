---
title: Vertical Slice Cheatsheet
date: 2025-11-11
tags: 
    - spring
    - java
    - architecture
    - software-design
    - vertical-slice
summary: A practical cheatsheet for building vertical slices in Spring Boot applications.
aliases:
    - Spring Vertical Slice Cheatsheet
---

# Vertical Slice — Spring Boot Cheatsheet

Zoom in on “vertical slices.” It’s not just a buzzword—it’s a practical way to ship value and keep complexity fenced in. Here’s a tight cheatsheet you can paste into your vault and actually use while coding.

## What a “vertical slice” is (in practice)

A **self-contained feature** that cuts straight through the stack: HTTP contract → web edge (DTOs/validation/errors) → app/use-case → domain/rules → persistence/migration → tests → observability. It’s small, shippable, and independently testable.

**Litmus test:** If you can delete this slice with minimal collateral damage, you sliced correctly.

---

## Success criteria (use this as a gate)

* Delivers a user-visible outcome (not just plumbing).
* Has a clear API contract (request/response + error cases).
* Owns its data shape (migrations included).
* Enforces invariants where they belong (domain + DB).
* Has fast tests across layers (unit, slice, one integration).
* Produces useful logs/metrics for that feature.
* No leaks: controllers don’t carry domain logic; entities don’t know HTTP.

---

## The 8-step “slice loop”

1. **Name the story**: “Create category with unique name.”
2. **Define the contract** (examples first):

    * `POST /api/categories`
    
        Request: `{ "name": "Books" }`
      
        Response: `201 Created` + `Location: /api/categories/{id}`
   
        Errors: `409` type `/problems/duplicate-category`

3. **Draft acceptance & edge cases**:

    * Trim + non-blank name, normalize uniqueness (`lowercase`).

4. **Design the data**:

    * Flyway: new table/index or change; write Vxxx migration.

5. **Model the domain**:

    * Aggregate/value objects; invariants in code.

6. **Wire the app/use-case**:

    * One handler/service per story; orchestrate repo calls.

7. **Web edge**:

    * DTOs (request/response), validation, ProblemDetail mapping, link/URI helpers.

8. **Tests & obs**:

    * Unit (domain), slice tests (`@DataJpaTest`, `@WebMvcTest`), one integration (Testcontainers), one log/metric.

Keep slices **thin**. If you touch more than ~6 files outside the feature’s package, you’re probably slicing horizontally.

---

## Minimal folder shape (feature-oriented, layered)

```
src/main/java/com/example/app/
  web/category/...
  app/category/...
  domain/category/...
  infra/{jpa,mapping,config}/...
src/main/resources/db/migration/V001__...
```

Repeat the same pattern per feature: `product`, `cart`, `order`, etc.

---

## Controller contract template

```java
// web/category/CategoryController.java
@RestController
@RequestMapping("/api/categories")
class CategoryController {

  private final CreateCategoryHandler create;
  private final CategoryQueries queries;

  CategoryController(CreateCategoryHandler create, CategoryQueries queries) {
    this.create = create; this.queries = queries;
  }

  @PostMapping
  ResponseEntity<Void> create(@Valid @RequestBody CreateCategoryRequest req, UriComponentsBuilder uris) {
    var id = create.handle(req.name());
    return ResponseEntity.created(uris.path("/api/categories/{id}").build(id)).build();
  }

  @GetMapping
  Page<CategoryResponse> list(@PageableDefault(size=20) Pageable p) {
    return queries.list(p);
  }
}
```

---

## ProblemDetail pattern (global)

```java
@RestControllerAdvice
class GlobalAdvice {
  @ExceptionHandler(DuplicateCategoryException.class)
  ProblemDetail duplicate(DuplicateCategoryException ex, HttpServletRequest req) {
    var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setTitle("Duplicate category");
    pd.setType(URI.create("/problems/duplicate-category"));
    pd.setDetail("Category already exists: " + ex.normalized());
    pd.setProperty("instance", req.getRequestURI());
    return pd;
  }
}
```

---

## Testing recipe (copy this list into each PR)

* **Domain unit:** invariants (trim/blank/normalize).
* **Repo slice (`@DataJpaTest`):** unique index behavior.
* **Web slice (`@WebMvcTest`):** 201 + Location; 409 ProblemDetail shape.
* **Integration:** boot + Testcontainers → POST then GET happy path.

> Rule: at least **3 fast tests + 1 integration** per slice.

---

## Observability checklist (per slice)

* Log one business event: `category.created` with `id` and `normalizedName`.
* Metric or counter if meaningful: `categories_created_total`.
* Add a trace tag if you’ve enabled OpenTelemetry.

---

## DoD for a slice (Definition of Done)

* ✅ Migration merged and applied locally.
* ✅ API returns correct status codes & ProblemDetail.
* ✅ Tests green: unit, slice, integration.
* ✅ Logs/metrics wired.
* ✅ No stringly-typed URIs left in controllers (use a helper).
* ✅ README/changelog entry for the feature (2–3 lines).

---

## Common pitfalls to dodge

* **Horizontal slicing:** building “the repository layer” first. Always ship a user story.
* **Leaky controllers:** validation/HTTP in web; rules in domain/app.
* **Letting JPA design your schema:** write migrations first.
* **Skipping idempotency** on commands that can be retried (e.g., checkout, add-to-cart).
* **Too many refactors after surveying:** apply only the top 1–2 improvements; park the rest.

---

## Branch/PR hygiene

* One branch per slice, ≤ 300–500 LOC.
* PR includes: migration, code, tests, short notes on decisions.
* CI runs unit + slice + integration tests.

---

## Tiny example: the Category slice checklist

* Contract written with examples.
* `V001__categories.sql` with `unique(normalized_name)`.
* `Category` domain with normalization.
* Repo with `existsByNormalizedName`.
* `CreateCategoryHandler` use-case.
* DTOs + Controller + ProblemDetail mapping.
* Tests: unit + JPA + WebMvc + integration.
* Log `category.created`; counter optional.

---

## When to add security to a slice

* After 1–2 slices are working, protect **mutating endpoints** (create/update/delete) and leave reads public.
* Add one security test per protected endpoint.

---

## Refactor cadence (so you don’t stall)

* After a slice, run a **30-minute survey** (docs/samples).
* Apply **max two** refactors.
* Record the rest in `research-debt.md` with a date.

---

Treat this like a flight checklist. Run it verbatim for “CreateCategory,” then again for “CreateProduct” (with pagination/filtering), then “AddToCart” (idempotent add). After two or three passes, the pattern becomes muscle memory and you’ll stop second-guessing the “perfect path.”
