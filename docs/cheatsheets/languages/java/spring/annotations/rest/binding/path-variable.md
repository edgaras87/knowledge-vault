---
title: "@PathVariable"
date: 2025-10-22
tags:
    - spring
    - java
    - spring-mvc
    - rest
    - web
    - annotations
    - cheatsheet
summary: binding URL path segments to controller method parameters, with patterns, type conversion, regex constraints, catch-alls, and best practices.
aliases:
    - Spring @PathVariable Annotation Cheatsheet
---

# 🧭 `@PathVariable` — URL Path Segments

**What it does:** Binds **URL template variables** from the path to controller method parameters.

Use it when the variable is **part of the resource identity**: `/users/{id}`, `/shops/{shopId}/orders/{orderId}`. For query filters or toggles, prefer `@RequestParam`.

---

## 1) Daily patterns

```java
// /users/42
@GetMapping("/users/{id}")
public UserDto get(@PathVariable long id) { /* ... */ }

// Multiple variables: /shops/7/orders/99
@GetMapping("/shops/{shopId}/orders/{orderId}")
public OrderDto get(@PathVariable long shopId, @PathVariable long orderId) { /* ... */ }

// Rename the variable: /files/2025-10/report.pdf
@GetMapping("/files/{fileName}")
public FileDto get(@PathVariable("fileName") String name) { /* ... */ }

// Grab them all (debuggy): /echo/anything/here
@GetMapping("/echo/{x}/{y}")
public Map<String,String> echo(@PathVariable Map<String,String> vars) { return vars; }
```

---

## 2) Type conversion (built-in + custom)

Spring converts strings from the path via the **ConversionService**.

Works out-of-the-box for: primitives/wrappers, `UUID`, enums, `LocalDate/LocalDateTime`, `BigDecimal`, etc.

```java
// /invoices/550e8400-e29b-41d4-a716-446655440000
@GetMapping("/invoices/{id}")
public InvoiceDto get(@PathVariable UUID id) { /* ... */ }

// Custom type
@Component
class SlugToProductId implements Converter<String, ProductId> {
  public ProductId convert(String s) { return ProductId.of(s.toLowerCase()); }
}

@GetMapping("/products/{pid}")
public ProductDto get(@PathVariable ProductId pid) { /* ... */ }
```

---

## 3) Constraining with regex

Lock paths down to valid shapes:

```java
// digits only: /users/123
@GetMapping("/users/{id:\\d+}")

// year-month: /reports/2025-10
@GetMapping("/reports/{ym:\\d{4}-\\d{2}}")

// enum-like set: /status/ACTIVE
@GetMapping("/status/{s:ACTIVE|SUSPENDED}")
```

Regex is per-segment (no slashes unless you use a catch-all—see below).

---

## 4) Catch-all (include slashes)

Sometimes you need “the rest of the path,” e.g., serving files under a base.

* **AntPathMatcher style (older config):** `"{path:**}"`
* **PathPatternParser style (modern Spring/Boot 3+):** `"{*path}"`

```java
// /raw/a/b/c.txt  → path = "a/b/c.txt"
@GetMapping("/raw/{*path}")              // If PathPatternParser is enabled (default in recent Boot)
public Resource raw(@PathVariable String path) { /* ... */ }

// Fallback if using AntPathMatcher:
@GetMapping("/raw/{path:**}")
public Resource rawLegacy(@PathVariable("path") String path) { /* ... */ }
```

---

## 5) Optional path variables (the right way)

Path variables are **required by default**. To make a segment optional, provide **two mappings** and mark the parameter optional.

```java
// /users           → id = null
// /users/42        → id = 42
@GetMapping({"/users", "/users/{id}"})
public List<UserDto> list(@PathVariable(required = false) Long id) { /* ... */ }
```

There’s no native “optional segment” syntax in a single pattern for MVC; use multiple paths.

---

## 6) Dots, dashes, spaces & URL decoding

* Spring decodes `%20` → space, `%2B` → `+`, etc.
* **Dots (`.`)** are safe in recent Spring Boot defaults (suffix pattern matching is off by default). If you still see truncation after a dot, ensure you’re using the modern **PathPatternParser** and not legacy suffix pattern matching.
* A single `{var}` **never includes slashes**. Use a catch-all to include `/`.

---

## 7) Validation & error handling

If binding fails:

* Missing variable → `MissingPathVariableException` (500 if your method signature requires it but mapping doesn’t define it; usually a route/config bug).
* Type mismatch → `MethodArgumentTypeMismatchException` (400).

Centralize:

```java
@RestControllerAdvice
class ApiErrors {
  @ExceptionHandler(MethodArgumentTypeMismatchException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  ErrorDto badPath(MethodArgumentTypeMismatchException ex) {
    return new ErrorDto("BAD_PATH", ex.getMessage());
  }
}
```

You can add Bean Validation to **converted types** if you bind to a record/POJO via `@ModelAttribute` (less common for pure path vars).

---

## 8) Versioning & hierarchy examples

```java
// Identity first, then sub-resource
@GetMapping("/users/{id}/addresses/{addrId}")
public AddressDto get(@PathVariable long id, @PathVariable long addrId) { /* ... */ }

// API version in the path
@GetMapping("/v1/users/{id}")
public UserDto v1(@PathVariable long id) { /* ... */ }
```

---

## 9) `@PathVariable` vs friends

* **`@PathVariable`** — identity in the URL: `/users/{id}`.
* **`@RequestParam`** — filters/sorting/pagination: `/users?page=2&active=true`.
* **`@RequestBody`** — JSON body for create/update.
* **`@MatrixVariable`** — semi-colon parameters in a path segment (`/cars;color=red;year=2025`) when matrix vars are enabled.

---

## 10) Trailing slashes, case, and matching engine

* **Trailing slash**: `/users/42/` vs `/users/42` — by default, treated differently. Add both mappings or configure to be tolerant.
* **Case sensitivity**: paths are case-sensitive by default.
* **Matching engine**: Modern Spring MVC favors **PathPatternParser** (faster, more precise). It changes catch-all syntax (`{*var}`) compared to the older Ant style (`{var:**}`). Pick one and stick with it across your project.

---

## 11) Minimal idiomatic set

```java
// 1) Simple identity
@GetMapping("/items/{id}")
public ItemDto one(@PathVariable long id) { /* ... */ }

// 2) Nested resource
@GetMapping("/shops/{shopId}/items/{itemId}")
public ItemDto shopItem(@PathVariable long shopId, @PathVariable long itemId) { /* ... */ }

// 3) Regex guard
@GetMapping("/tickets/{num:\\d+}")
public TicketDto ticket(@PathVariable int num) { /* ... */ }

// 4) Catch-all
@GetMapping("/assets/{*path}")
public Resource asset(@PathVariable String path) { /* ... */ }
```

---

## 12) Quick reference table

| Need                   | Pattern                                         | Example           |
| ---------------------- | ----------------------------------------------- | ----------------- |
| Single segment         | `/{id}`                                         | `/users/42`       |
| Rename arg             | `@PathVariable("userId") Long id`               | `/users/42`       |
| Multiple vars          | `/{a}/{b}`                                      | `/a/10/b/20`      |
| Regex constraint       | `/{id:\\d+}`                                    | `/users/123` only |
| Catch-all (modern)     | `/{*path}`                                      | `/raw/a/b/c.txt`  |
| Catch-all (legacy Ant) | `/{path:**}`                                    | `/raw/a/b/c.txt`  |
| Optional segment       | `@GetMapping({"/u", "/u/{id}"})`                | `/u` or `/u/7`    |
| Map of vars            | `@PathVariable Map<String,String>`              | `/x/1/y/2`        |
| Type conversion        | `UUID`, `Enum`, `LocalDate`, custom `Converter` | `/inv/uuid`       |

---

### Mental model

`@PathVariable` is about **identity baked into the URL**. It’s segment-based, decoded to types, and happiest when the path shape is **explicit**: constrain with regex when it helps, and use catch-alls only where they truly make sense.

