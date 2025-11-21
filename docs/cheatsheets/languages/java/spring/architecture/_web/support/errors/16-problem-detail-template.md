---
title: ProblemDetail Templates
date: 2025-11-21
tags: 
    - spring
    - spring-boot
    - spring-web
    - rest-api
    - error-handling
    - problem-detail
  
summary: Minimal cheatsheet & templates for Spring's ProblemDetail error representation.
aliases:
  - ProblemDetail Templates

---




# ProblemDetail – Minimal Cheatsheet & Templates

## Contents

1. [What it is](#1-what-it-is)  
2. [GlobalExceptionAdvice – base skeleton](#2-globalexceptionadvice--base-skeleton)  
3. [Template – custom domain/app exception → ProblemDetail](#3-template--custom-domainapp-exception--problemdetail)  
  3.1 [Simple, explicit version](#31-simple-explicit-version)  
  3.2 [Version with small helper factory](#32-version-with-small-helper-factory)  
4. [Templates – Jakarta Bean Validation errors](#4-templates--jakarta-bean-validation-errors)  
  4.1 [`MethodArgumentNotValidException` (request body @Valid) – field → message map](#41-methodargumentnotvalidexception-request-body-valid--field--message-map)  
  4.2 [`MethodArgumentNotValidException` – simple list of messages](#42-methodargumentnotvalidexception--simple-list-of-messages)  
  4.3 [`ConstraintViolationException` (e.g. @Validated on params)](#43-constraintviolationexception-e-g--validated-on-params)  


## 1. What it is

```java
import org.springframework.http.ProblemDetail;
```

Typical JSON shape:

```json
{
  "type":   "https://api.example.com/problems/duplicate-category",
  "title":  "Category already exists",
  "status": 409,
  "detail": "Category with slug 'electronics' already exists",
  "instance": "/api/categories"
}
```

**Core methods (1–line mental model)**

* `ProblemDetail.forStatus(HttpStatus)` – create with HTTP status.
* `ProblemDetail.forStatus(int)` – create with raw status code.
* `setTitle(String)` – short, reusable name of this problem type.
* `setDetail(String)` – concrete message for this specific occurrence.
* `setType(URI)` – stable URI identifying the problem kind.
* `setInstance(URI)` – URI of the request that failed.

**Custom properties**

* `setProperty(String key, Object value)` – add extra JSON fields (e.g. `code`, `errors`, `timestamp`).

---

## 2. GlobalExceptionAdvice – base skeleton {#2-globalexceptionadvice--base-skeleton}

```java
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionAdvice {

    // Helper: current request URI as instance
    private static URI instanceOf(HttpServletRequest request) {
        return URI.create(request.getRequestURI());
    }

    // Handlers below...
}
```

---

## 3. Template – custom domain/app exception → ProblemDetail {#3-template--custom-domainapp-exception--problemdetail}

### 3.1 Simple, explicit version

```java
@ExceptionHandler(DuplicateCategoryException.class)
ProblemDetail handleDuplicateCategory(DuplicateCategoryException ex,
                                      HttpServletRequest request) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);

    pd.setTitle("Category already exists"); // stable, short label
    pd.setDetail(ex.getMessage());          // per-request detail
    pd.setType(URI.create("https://api.example.com/problems/duplicate-category"));
    pd.setInstance(instanceOf(request));

    pd.setProperty("code", "duplicate-category"); // machine-friendly code
    pd.setProperty("slug", ex.getSlug());         // extra context
    pd.setProperty("timestamp", Instant.now());   // optional

    return pd;
}
```

### 3.2 Version with small helper factory

```java
// Optional helper inside GlobalExceptionAdvice
private ProblemDetail problem(HttpStatus status,
                              String title,
                              String code,
                              URI type,
                              String detail,
                              HttpServletRequest request) {
    ProblemDetail pd = ProblemDetail.forStatus(status);
    pd.setTitle(title);
    pd.setDetail(detail);
    pd.setType(type);
    pd.setInstance(instanceOf(request));
    if (code != null) {
        pd.setProperty("code", code);
    }
    return pd;
}

// Handler using the helper
@ExceptionHandler(DuplicateCategoryException.class)
ProblemDetail handleDuplicateCategory2(DuplicateCategoryException ex,
                                       HttpServletRequest request) {
    ProblemDetail pd = problem(
            HttpStatus.CONFLICT,
            "Category already exists",
            "duplicate-category",
            URI.create("https://api.example.com/problems/duplicate-category"),
            ex.getMessage(),
            request
    );

    pd.setProperty("slug", ex.getSlug()); // add extra properties per case
    return pd;
}
```

---

## 4. Templates – Jakarta Bean Validation errors {#4-templates--jakarta-bean-validation-errors}

### 4.1 `MethodArgumentNotValidException` (request body @Valid) – field → message map {#41-methodargumentnotvalidexception-request-body-valid--field--message-map}

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
ProblemDetail handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
                                           HttpServletRequest request) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);

    pd.setTitle("Request validation failed");
    pd.setDetail("Validation failed for one or more fields.");
    pd.setType(URI.create("https://api.example.com/problems/validation-failed"));
    pd.setInstance(instanceOf(request));
    pd.setProperty("code", "validation-failed");

    Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                    fieldError -> fieldError.getField(),
                    fieldError -> fieldError.getDefaultMessage(),
                    (first, ignored) -> first // keep first message per field
            ));

    pd.setProperty("errors", errors); // { "field": "message", ... }

    return pd;
}
```

### 4.2 `MethodArgumentNotValidException` – simple list of messages {#42-methodargumentnotvalidexception--simple-list-of-messages}

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
ProblemDetail handleMethodArgumentNotValidSimple(MethodArgumentNotValidException ex,
                                                 HttpServletRequest request) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);

    pd.setTitle("Request validation failed");
    pd.setDetail("Request body is invalid.");
    pd.setType(URI.create("https://api.example.com/problems/validation-failed"));
    pd.setInstance(instanceOf(request));
    pd.setProperty("code", "validation-failed");

    List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .toList();

    pd.setProperty("errors", errors); // [ "field: message", ... ]

    return pd;
}
```

### 4.3 `ConstraintViolationException` (e.g. @Validated on params) {#43-constraintviolationexception-e-g--validated-on-params}

```java
import jakarta.validation.ConstraintViolationException;

@ExceptionHandler(ConstraintViolationException.class)
ProblemDetail handleConstraintViolation(ConstraintViolationException ex,
                                        HttpServletRequest request) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);

    pd.setTitle("Constraint violation");
    pd.setDetail("Validation failed for request parameters.");
    pd.setType(URI.create("https://api.example.com/problems/constraint-violation"));
    pd.setInstance(instanceOf(request));
    pd.setProperty("code", "constraint-violation");

    List<String> errors = ex.getConstraintViolations()
            .stream()
            .map(v -> v.getPropertyPath() + ": " + v.getMessage())
            .toList();

    pd.setProperty("errors", errors);

    return pd;
}
```

---

That’s basically your “muscle memory kit”: copy one of each handler style, swap the exception type, tweak `status/title/type/code/detail`, and you’re done. Over time you’ll just pattern-match “ah yeah, this is the 409 template with a different code”.
