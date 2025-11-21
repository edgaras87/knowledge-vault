---
title: ProblemDetail
date: 2025-11-20
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - errors
    - cheatsheet 
  
summary: A practical cheatsheet on using Spring's ProblemDetail for standardized error responses in Java Spring applications, detailing best practices and providing real-world examples.
aliases:
  - ProblemDetail Cheatsheet

---



# Using Spring's `ProblemDetail` for Standardized Error Responses

---

# Table of Contents

1. [What is `ProblemDetail` exactly?](#1-what-is-problemdetail-exactly)
2. [Why use `ProblemDetail` instead of your own error DTO?](#2-why-use-problemdetail-instead-of-your-own-error-dto)
3. [How to create one](#3-how-to-create-one)
4. [Typical Spring Boot usage pattern](#4-typical-spring-boot-usage-pattern)
5. [How `ProblemDetail` interacts with Spring’s internals](#5-how-problemdetail-interacts-with-springs-internals)
6. [Where do you define problem URIs (`type`)?](#6-where-do-you-define-problem-uris-type)
7. [Folder / responsibility placement (in your layered style)](#7-folder--responsibility-placement-in-your-layered-style)
8. [What about validation errors?](#8-what-about-validation-errors)
9. [Mental model to keep](#9-mental-model-to-keep)


---

## 1. What is `ProblemDetail` exactly?

Classpath:

```java
import org.springframework.http.ProblemDetail;
```

It’s a **POJO representing a problem** as defined by RFC 7807. The typical JSON shape:

```json
{
  "type":   "https://api.example.com/problems/duplicate-category",
  "title":  "Category already exists",
  "status": 409,
  "detail": "Category with slug 'electronics' already exists",
  "instance": "/api/categories"
}
```

Core fields:

* `type`: URI identifying the kind of problem (like an error code but as a URI).
* `title`: Short, human-readable title (same for that type).
* `status`: HTTP status code (int).
* `detail`: Human-readable description specific to this occurrence.
* `instance`: URI of the specific request instance (e.g., `/api/categories/123`).

Spring’s `ProblemDetail` also lets you attach **custom properties** in the JSON via a map, e.g. `code`, `errors`, etc.

---

## 2. Why use `ProblemDetail` instead of your own error DTO?

You *could* just do this:

```java
record ErrorResponse(String message, Instant timestamp) {}
```

But:

1. **Consistent format**
   RFC 7807 is a standard. Many clients/tools already expect `type/title/status/detail/instance`.

2. **Built into Spring**
   `ProblemDetail` integrates with:

   * `@ExceptionHandler`
   * `ResponseStatusException`
   * New built-in exception resolvers
     So you get auto-mapping, sensible defaults, less boilerplate.

3. **Extensible but predictable**
   You have a fixed core shape, plus freeform extra fields. You can add:

   ```json
   {
     "code": "duplicate-category",
     "errors": {
       "name": "must be unique within parent category"
     }
   }
   ```

---

## 3. How to create one

Basic creation:

```java
ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
pd.setTitle("Category not found");
pd.setDetail("Category with id 42 not found");
```

Or with raw status code:

```java
ProblemDetail pd = ProblemDetail.forStatus(404);
```

Setting the `type` and `instance`:

```java
pd.setType(URI.create("https://api.example.com/problems/category-not-found"));
pd.setInstance(URI.create("/api/categories/42"));
```

Adding custom fields:

```java
pd.setProperty("code", "category-not-found");
pd.setProperty("timestamp", Instant.now());
pd.setProperty("errors", Map.of("id", "Unknown category id"));
```

Spring will serialize all of that to JSON.

---

## 4. Typical Spring Boot usage pattern

### 4.1. Throw domain/app exceptions in services

Example domain exception:

```java
public class DuplicateCategoryException extends RuntimeException {
    private final String slug;

    public DuplicateCategoryException(String slug) {
        super("Category with slug '%s' already exists".formatted(slug));
        this.slug = slug;
    }

    public String getSlug() {
        return slug;
    }
}
```

Service:

```java
@Service
public class CreateCategoryHandler {

    private final CategoryRepository repository;

    public CreateCategoryHandler(CategoryRepository repository) {
        this.repository = repository;
    }

    public Category handle(CreateCategoryCommand command) {
        if (repository.existsBySlug(command.slug())) {
            throw new DuplicateCategoryException(command.slug());
        }

        Category category = new Category(command.name(), command.slug());
        return repository.save(category);
    }
}
```

### 4.2. Translate once at the web edge with `@RestControllerAdvice`

```java
@RestControllerAdvice
public class GlobalExceptionAdvice {

    @ExceptionHandler(DuplicateCategoryException.class)
    public ProblemDetail handleDuplicateCategory(DuplicateCategoryException ex,
                                                 HttpServletRequest request) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);

        pd.setTitle("Category already exists");
        pd.setDetail(ex.getMessage());
        pd.setType(URI.create("https://api.example.com/problems/duplicate-category"));

        // Request path as instance
        pd.setInstance(URI.create(request.getRequestURI()));

        // Custom properties for machines
        pd.setProperty("code", "duplicate-category");
        pd.setProperty("slug", ex.getSlug());

        return pd;
    }
}
```

Controller method stays clean:

```java
@PostMapping("/api/categories")
public ResponseEntity<CategoryResponse> createCategory(@RequestBody CreateCategoryRequest request) {
    Category created = handler.handle(new CreateCategoryCommand(request.name(), request.slug()));
    // map to response, build Location header, etc.
}
```

If something goes wrong → Spring sees that the `@ExceptionHandler` returns `ProblemDetail` → serializes it.

---

## 5. How `ProblemDetail` interacts with Spring’s internals

Some key points:

* Returning `ProblemDetail` from controller or `@ExceptionHandler` = **direct response body**.

* `ResponseStatusException` in Spring 6 can **carry** a `ProblemDetail`. For example:

  ```java
  ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
  pd.setTitle("Invalid request");
  throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Invalid request", new MyException())
          .withDetail(pd); // pattern varies by version, idea is: RSE + PD
  ```

  (Exact API sugar depends slightly on Spring version, but the concept: RSE and ProblemDetail can interact.)

* Some Spring components (validation, etc.) can produce `ProblemDetail` automatically if you configure the error handling to do so, but **for real control you usually write your own `@RestControllerAdvice`**.

---

## 6. Where do you define problem URIs (`type`)?

This was your earlier question:

> “What should live on those URIs like `https://api.example.com/problems/duplicate-category`?”

Reality:

* Most teams don’t host a giant HTML page there.
* Minimal, pragmatic approach: host **a tiny human-readable description** or even just plain text/JSON with:

  * name
  * explanation
  * maybe example payload.

For practice:

* Using fake-but-stable URIs is fine:

  ```java
  URI.create("https://test.local/problems/duplicate-category");
  ```
* In a real app, you often **externalize** the base:

  ```yaml
  app:
    problems:
      base: https://api.example.com/problems/
  ```

  Then:

  ```java
  @ConfigurationProperties(prefix = "app.problems")
  public record ProblemProps(URI base) {}
  ```

  And in your advice:

  ```java
  pd.setType(problemProps.base().resolve("duplicate-category"));
  ```

That way, if the base changes once, you fix it in properties, not in 50 handlers.

---

## 7. Folder / responsibility placement (in your layered style)

Given your `web/app/domain/infra` idea:

* `web/support/errors/GlobalExceptionAdvice.java`
  → All the `@ExceptionHandler` logic returning `ProblemDetail`.

* Optional helpers:

  * `web/support/errors/ProblemTypes.java` – constants for `URI`s.
  * `web/support/errors/ProblemCodes.java` – short string codes.
  * `web/support/errors/ProblemDetailFactory.java` – factory methods if you prefer not to repeat.

Example helper:

```java
public final class ProblemTypes {
    private ProblemTypes() {}

    public static final URI DUPLICATE_CATEGORY =
            URI.create("https://api.example.com/problems/duplicate-category");

    public static final URI VALIDATION_FAILED =
            URI.create("https://api.example.com/problems/validation-failed");
}
```

Usage:

```java
pd.setType(ProblemTypes.DUPLICATE_CATEGORY);
```

This keeps the `GlobalExceptionAdvice` readable and gives you one place to see all problems.

---

## 8. What about validation errors?

`ProblemDetail` plays nicely with bean validation:

Typical pattern:

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ProblemDetail handleValidation(MethodArgumentNotValidException ex,
                                      HttpServletRequest request) {

    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Request validation failed");
    pd.setType(URI.create("https://api.example.com/problems/validation-failed"));
    pd.setInstance(URI.create(request.getRequestURI()));

    Map<String, String> errors = ex.getBindingResult()
        .getFieldErrors()
        .stream()
        .collect(Collectors.toMap(
            FieldError::getField,
            DefaultMessageSourceResolvable::getDefaultMessage,
            (oldVal, newVal) -> oldVal // in case of duplicate keys
        ));

    pd.setProperty("code", "validation-failed");
    pd.setProperty("errors", errors);

    return pd;
}
```

Client now gets:

```json
{
  "type": "https://api.example.com/problems/validation-failed",
  "title": "Request validation failed",
  "status": 400,
  "detail": "Validation failed for object='createCategoryRequest'. Error count: 1",
  "instance": "/api/categories",
  "code": "validation-failed",
  "errors": {
    "name": "must not be blank",
    "slug": "must match ^[a-z0-9-]+$"
  }
}
```

---

## 9. Mental model to keep

When you see `ProblemDetail`, read it as:

> *“One structured description of a failure for this HTTP request, with a globally identifiable problem type and a machine-friendly payload.”*

The flow in your head:

* Domain/app layer throws exception (`DuplicateCategoryException`).
* Web edge catches it once.
* Maps it to `ProblemDetail` with:

  * `type` = which kind of problem.
  * `detail`/`title` = human text.
  * `code`/`errors`/etc. = machine data.
* Client uses `type` + extra fields to decide what to do.

