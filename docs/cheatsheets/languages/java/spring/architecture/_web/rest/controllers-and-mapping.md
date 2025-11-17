---
title: "Spring Controller Annotations & Mapping"
date: 2025-10-22
tags:
    - spring
    - java
    - spring-mvc
    - web
    - rest
    - annotations
    - cheatsheet
summary: Concise reference for Spring MVC controllers and request mapping annotations, covering stereotypes, mapping attributes, data binding, return types, error handling, and practical recipes.
aliases:
    - Spring Controller Annotations
---

# 🚦 Controllers & Request Mapping — The Practical Map

## 0) Stereotypes (what the class is)

```java
@Controller             // Returns views (templates); methods default to view names.
@RestController         // = @Controller + @ResponseBody on every method (JSON by default).
```

* Use `@RestController` for APIs.
* If you need both views and JSON in one class, stick to `@Controller` and annotate JSON methods with `@ResponseBody`.

---

## 1) Mapping annotations (what path/verb hits the method)

**The base:**

```java
@RequestMapping(
  path = "/users",      // or "value"
  method = GET,         // or multiple: {GET, HEAD}
  consumes = "application/json",
  produces = "application/json",
  params = "active=true",     // require or negate (e.g., "!debug")
  headers = "X-Api-Version=1" // same idea for headers
)
```

**Composed, ergonomic variants (internally `@RequestMapping(method=...)`):**

```java
@GetMapping("/users/{id}")        // GET
@PostMapping("/users")            // POST
@PutMapping("/users/{id}")        // PUT
@PatchMapping("/users/{id}")      // PATCH
@DeleteMapping("/users/{id}")     // DELETE
```

**Class + method composition:**

```java
@RestController
@RequestMapping("/v1/users")      // base path, base produces/consumes can live here
class UserApi {

  @GetMapping("/{id}")            // → /v1/users/{id}
  UserDto get(@PathVariable long id) { ... }

  @PostMapping(consumes="application/json")
  UserDto create(@RequestBody CreateUser dto) { ... }
}
```

---

## 2) Mapping attributes — quick semantics

| Attribute        | Where                       | What it does                    | Example                                       |
| ---------------- | --------------------------- | ------------------------------- | --------------------------------------------- |
| `path` / `value` | class/method                | URL template(s)                 | `"/orders/{id}"`, `{"","/list"}`              |
| `method`         | method or `@RequestMapping` | HTTP verb(s)                    | `{GET, HEAD}`                                 |
| `consumes`       | method/class                | Require content type of request | `"application/json"`, `"multipart/form-data"` |
| `produces`       | method/class                | Content type of response        | `"application/json"`, `"text/csv"`            |
| `params`         | method/class                | Require/forbid query param(s)   | `"mode=debug"`, `"!page"`                     |
| `headers`        | method/class                | Require/forbid header(s)        | `"X-Auth-Token"`                              |

**Notes**

* `produces` participates in **content negotiation**; Spring also checks `Accept` header.
* `params/headers` are underrated: use them to separate HTML vs JSON for same path if needed.

---

## 3) Bind incoming data (arguments you’ll actually use)

```java
@PathVariable Long id                // /items/{id}
@RequestParam Integer page           // ?page=2 (query or form)
@RequestParam(defaultValue="20") int size
@RequestHeader("X-Trace") String traceId
@CookieValue("sid") String session
@RequestBody OrderCreate dto         // JSON/XML/etc.
@ModelAttribute Filter f             // group simple params into a bean (query/form)
@RequestPart("file") MultipartFile f // explicit part name in multipart
```

* For full details, see your `binding/` cheatsheets, but this table helps decide fast:

| You have…                      | Use                                            | Example               |
| ------------------------------ | ---------------------------------------------- | --------------------- |
| Identifier in the path         | `@PathVariable`                                | `/users/{id}`         |
| Filter/sort/toggle             | `@RequestParam`                                | `?active=true&page=2` |
| Headers/cookies                | `@RequestHeader` / `@CookieValue`              | `X-Trace`, `sid`      |
| JSON body                      | `@RequestBody`                                 | `POST {…}`            |
| Many simple params as one bean | `@ModelAttribute`                              | `Filter{q,page,size}` |
| Multipart upload               | `@RequestParam MultipartFile` / `@RequestPart` | file forms            |

---

## 4) Return types (what you send back)

```java
// JSON (in @RestController this is default):
UserDto                       // body is the object, status 200
ResponseEntity<UserDto>       // control status/headers/body
void                          // often with 204 or set via @ResponseStatus
String                        // in @Controller → view name; in @RestController → plain text
```

Helpful extras:

```java
@ResponseStatus(HttpStatus.CREATED)
@PostMapping("/users")
UserDto create(@RequestBody CreateUser dto) { ... }

@GetMapping("/health")
ResponseEntity<Void> ok() { return ResponseEntity.noContent().build(); }
```

---

## 5) Cross-cutting web annotations that matter

```java
@CrossOrigin(origins="https://app.example.com")  // CORS at method/class level
@ExceptionHandler(MyAppException.class)          // handle errors in this controller
@ControllerAdvice                                // global error/advice across controllers
@InitBinder                                      // register binders/validators for params
```

* Prefer a **global** `@RestControllerAdvice` with typed handlers to shape your API errors.

---

## 6) Practical recipes

**A) CRUD slice with good defaults**

```java
@RestController
@RequestMapping(path="/v1/items", produces="application/json")
class ItemsApi {

  @GetMapping("/{id}")
  ItemDto one(@PathVariable long id) { ... }

  @GetMapping
  PageDto<ItemDto> list(
      @RequestParam(defaultValue="1") int page,
      @RequestParam(defaultValue="20") int size,
      @RequestParam(required=false) String q) { ... }

  @PostMapping(consumes="application/json")
  @ResponseStatus(HttpStatus.CREATED)
  ItemDto create(@RequestBody CreateItem dto) { ... }

  @PutMapping(path="/{id}", consumes="application/json")
  ItemDto replace(@PathVariable long id, @RequestBody UpdateItem dto) { ... }

  @DeleteMapping("/{id}")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  void delete(@PathVariable long id) { ... }
}
```

**B) Same path, different `produces` (HTML vs JSON)**

```java
@Controller
@RequestMapping("/reports")
class Reports {

  @GetMapping(produces="text/html")
  String page(Model m) { /* put attrs → m */ return "reports"; }

  @GetMapping(produces="application/json")
  @ResponseBody
  List<ReportDto> api() { ... }
}
```

**C) Distinguish by params (classic)**

```java
@GetMapping(value="/search", params="mode=fast")
List<Result> fast(@RequestParam String q) { ... }

@GetMapping(value="/search", params="mode=deep")
List<Result> deep(@RequestParam String q, @RequestParam int limit) { ... }
```

**D) Multipart form (file + fields)**

```java
@PostMapping(path="/avatar", consumes=MediaType.MULTIPART_FORM_DATA_VALUE)
void upload(@RequestParam("file") MultipartFile file,
            @RequestParam(required=false) String note) { ... }
```

---

## 7) Matching engine, paths & regex

* Modern Spring (Boot 2.6+/3+) favors **PathPatternParser** (fast, precise).

  * Catch-all: `/{*path}`
  * Regex per segment: `"/tickets/{num:\\d+}"`
* Legacy Ant engine:

  * Catch-all: `/{path:**}`
* Avoid “suffix pattern matching”; keep dots (`.`) safe for names like `report.pdf`.

---

## 8) WebFlux notes (if you’re reactive)

* Same annotations and attributes.
* Return types are `Mono<T>` / `Flux<T>` (or `ServerSentEvent<T>` for SSE).
* `ResponseEntity<Mono<T>>` is valid, but idiomatic is `Mono<ResponseEntity<T>>`.

---

## 9) Error taxonomy you’ll see

| Situation                 | Exception                                                                  | Typical HTTP |
| ------------------------- | -------------------------------------------------------------------------- | ------------ |
| No route matched          | `NoHandlerFoundException` (if enabled)                                     | 404          |
| Missing path var/param    | `MissingPathVariableException` / `MissingServletRequestParameterException` | 400/500      |
| Type conversion fail      | `MethodArgumentTypeMismatchException`                                      | 400          |
| Unsupported media type    | `HttpMediaTypeNotSupportedException`                                       | 415          |
| Not acceptable (produces) | `HttpMediaTypeNotAcceptableException`                                      | 406          |

Centralize via:

```java
@RestControllerAdvice
class ApiErrors {
  @ExceptionHandler(MethodArgumentTypeMismatchException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  ErrorDto badArg(Exception e) { return new ErrorDto("BAD_REQUEST", e.getMessage()); }
}
```

---

## 10) Gotchas checklist (the foot-guns)

* Expecting CSV auto-split in `@RequestParam` → **won’t happen**; either repeat keys or split yourself.
* Using `@RequestParam` for JSON bodies → **use `@RequestBody`**.
* Optional primitives (`int`, `boolean`) → can’t be null. Use wrappers or defaults.
* Multiple mappings that collide (same path/verb/produces) → ambiguous mapping error.
* Forgetting `produces` while returning non-JSON in a `@RestController` → set it explicitly (e.g., CSV).
* Mixed engines (Ant vs PathPattern) → catch-all syntax differs; be consistent project-wide.

---

## 11) Ultra-compact reference tables

**A) Mapping annotations**

| Annotation                  | Verb   | Notes                        |
| --------------------------- | ------ | ---------------------------- |
| `@RequestMapping`           | any    | Full control over attributes |
| `@GetMapping`               | GET    | Idempotent retrieval         |
| `@PostMapping`              | POST   | Create/commands              |
| `@PutMapping`               | PUT    | Replace resource             |
| `@PatchMapping`             | PATCH  | Partial update               |
| `@DeleteMapping`            | DELETE | Remove resource              |
| `@RequestMapping(name=...)` | —      | Optional logical name        |

**B) Common attributes**

| Attribute      | Accepts         | Works with        |
| -------------- | --------------- | ----------------- |
| `path`/`value` | String/String[] | all               |
| `method`       | RequestMethod[] | `@RequestMapping` |
| `consumes`     | MediaType(s)    | all               |
| `produces`     | MediaType(s)    | all               |
| `params`       | String[]        | all               |
| `headers`      | String[]        | all               |

---

### Folder placement (fits your structure)

```
cheatsheets/frameworks/spring/web/
├─ controllers-and-mapping.md     # ← this file
├─ binding/                       # @PathVariable / @RequestParam / @RequestBody / overview
└─ routing/                       # (optional later) path patterns, CORS, interceptors
```

**Mental model to keep handy:**

> Map **who/where** with `@RequestMapping`/`@*Mapping`, bring data in with binding annotations, shape the response with a return type or `ResponseEntity`, and centralize errors with `@RestControllerAdvice`.

