---
title: "@RequestParam"
date: 2025-10-22
tags: 
    - spring
    - java
    - spring-mvc
    - rest
    - web
    - cheatsheet
summary: Comprehensive guide to using Spring's @RequestParam annotation for handling query and form parameters in RESTful APIs.
aliases:
    - Spring @RequestParam Annotation Cheatsheet
---

# ⚙️ `@RequestParam` — Query & Form Parameters

**What it does:** Binds **query string** parameters (`/api?q=java&page=2`) and **form fields** (`application/x-www-form-urlencoded` or simple `multipart/form-data`) to your controller method arguments.

**Remember:** `@RequestParam` is for *simple scalars & small structs*. For complex request bodies (JSON), use `@RequestBody`. For URL path pieces, use `@PathVariable`.

---

## 1) Fast patterns you’ll use daily

```java
@GetMapping("/search")
public List<BookDto> search(
    @RequestParam String q,                 // required by default
    @RequestParam(defaultValue = "1") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(required = false) String lang // nullable wrapper
) { /* ... */ }
```

```java
// Multi-values: ?tag=java&tag=spring
@GetMapping("/articles")
public List<ArticleDto> byTags(@RequestParam List<String> tag) { /* ... */ }
```

```java
// Optional without defaults
@GetMapping("/users")
public List<UserDto> users(@RequestParam Optional<String> email) { /* ... */ }
```

```java
// Arbitrary parameter bag: ?a=1&b=2
@GetMapping("/echo")
public Map<String, String> echo(@RequestParam Map<String, String> params) { /* ... */ }
```

```java
// MultiValueMap to preserve duplicates: ?k=1&k=2
@GetMapping("/kv")
public String kv(@RequestParam MultiValueMap<String, String> params) { /* ... */ }
```

---

## 2) Required vs default vs Optional

* **Required by default:** `@RequestParam String q` → 400 if missing.
* **Make optional:** `@RequestParam(required = false) String q` or use `Optional<String>`.
* **Provide a default:** `@RequestParam(defaultValue = "java") String q`

  * Setting `defaultValue` **implicitly makes it not required**.
  * With `Optional<T>` **and** `defaultValue`, you’ll get `Optional.of(default)`.

**Primitive vs wrapper:** prefer wrappers (`Integer`, `Boolean`) for optional params; primitives can’t be `null`.

---

## 3) Name inference & renaming

Spring infers parameter name from the Java arg name (if compiled with `-parameters`, which Spring Boot enables by default). To use a different query key:

```java
@GetMapping("/files")
public List<FileDto> list(@RequestParam("dir") String directory) { /* ... */ }
```

---

## 4) Multi-value binding

```java
// ?id=1&id=2&id=3
@GetMapping("/batch")
public List<UserDto> getBatch(@RequestParam List<Long> id) { /* ... */ }

// Arrays also work
public List<UserDto> getBatch(@RequestParam("id") long[] ids) { /* ... */ }

// CSV? Not automatic. Parse yourself:
@GetMapping("/batch2")
public List<UserDto> getBatchCsv(@RequestParam String idsCsv) {
  var ids = Arrays.stream(idsCsv.split(",")).map(Long::parseLong).toList();
  /* ... */
}
```

---

## 5) Supported sources (Spring MVC)

* **Query string:** `/api?q=java&page=1`
* **Form body:** `Content-Type: application/x-www-form-urlencoded`
* **Multipart fields:** alongside files in `multipart/form-data` (see below)

JSON bodies are **not** `@RequestParam`; use `@RequestBody`.

---

## 6) File upload quickies

```java
@PostMapping(path = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public UploadResult upload(
    @RequestParam("file") MultipartFile file,                 // single file
    @RequestParam(required = false) String note,
    @RequestParam("tags") List<String> tags                   // multi-values OK
) { /* ... */ }

// multiple files: @RequestParam("files") List<MultipartFile> files
```

---

## 7) Type conversion & custom formats

Spring converts strings → target types via the **ConversionService**.

Works out-of-the-box for: primitives, wrappers, enums, UUID, `LocalDate/LocalDateTime`, `BigDecimal`, etc. For custom types:

```java
@Component
public class SlugToProductIdConverter implements Converter<String, ProductId> {
  public ProductId convert(String s) { return ProductId.of(s.trim().toLowerCase()); }
}
```

Then:

```java
@GetMapping("/product")
public ProductDto product(@RequestParam ProductId id) { /* ... */ }
```

Dates:

```java
@GetMapping("/events")
public List<EventDto> events(
    @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate day
) { /* ... */ }
```

---

## 8) Validation

You can validate `@RequestParam` arguments with `@Validated` on the controller and JSR-380/Bean Validation annotations:

```java
@Validated
@RestController
class Api {
  @GetMapping("/offers")
  public List<OfferDto> offers(
      @RequestParam @Min(1) @Max(100) int size,
      @RequestParam @Pattern(regexp = "[a-z]{2}") String lang
  ) { /* ... */ }
}
```

For compound inputs, bind to a small DTO via `@ModelAttribute` (still query/form backed) and validate that object.

---

## 9) Empty vs missing vs default (edge-cases)

* **Missing** param (no key) + required → 400.
* **Empty present** (`?q=`) is **present** with `""`. With `required=true`, that’s fine; with `@NotBlank`, it will fail validation.
* **defaultValue** triggers when **missing** or **empty** (`""`), i.e., `?q=` yields default.

---

## 10) Encoding & weird characters

* Browsers encode spaces as `+` in query/form; Spring decodes to space. `%2B` is literal `+`.
* If a value needs commas/semicolons, encode them. Don’t rely on CSV unless you control the client.

---

## 11) `@RequestParam` vs the others

* **`@PathVariable`** — part of the URL structure: `/users/{id}`. Good for identity.
* **`@RequestParam`** — filters/toggles/pagination/search: `/users?active=true&page=2`.
* **`@RequestBody`** — JSON/XML payloads for create/update.
* **`@ModelAttribute`** — groups many simple params into a bean (query/form), supports validation.

---

## 12) Defaults that mirror product behavior

```java
@GetMapping("/list")
public PageDto list(
  @RequestParam(defaultValue = "1") int page,
  @RequestParam(defaultValue = "20") int size,
  @RequestParam(defaultValue = "name") String sortBy,
  @RequestParam(defaultValue = "asc") String order
) { /* ... */ }
```

This keeps your API forgiving while still deterministic.

---

## 13) Enum handling

```java
enum Status { ACTIVE, SUSPENDED }

@GetMapping("/accounts")
public List<AccountDto> list(@RequestParam Status status) { /* ?status=ACTIVE */ }

// Case-insensitive by default? No. Make a Converter or enable relaxed binding (WebDataBinder).
```

---

## 14) Error handling (missing/invalid params)

* Missing required → `MissingServletRequestParameterException` (400).
* Type mismatch → `MethodArgumentTypeMismatchException` (400).

Centralize with `@ControllerAdvice`:

```java
@RestControllerAdvice
class ApiErrors {
  @ExceptionHandler({ MissingServletRequestParameterException.class,
                      MethodArgumentTypeMismatchException.class })
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  public ErrorDto badRequest(Exception ex) {
    return new ErrorDto("BAD_REQUEST", ex.getMessage());
  }
}
```

---

## 15) Records / small DTOs with `@ModelAttribute`

When a method has a POJO/record parameter **without** `@RequestBody`, Spring treats it as a **model attribute** backed by query/form params:

```java
record Filter(String q, Integer page, Integer size) {}

@GetMapping("/search")
public List<ItemDto> search(Filter f) { /* /search?q=java&page=2 */ }
```

You can combine with `@Validated` on the type.

---

## 16) WebFlux notes (reactive)

Everything conceptually the same, but your method returns `Mono/Flux`. `@RequestParam` usage is identical; parameters are still decoded eagerly before the handler executes.

---

## 17) Common pitfalls checklist

* Using `@RequestParam` for JSON → don’t; use `@RequestBody`.
* Expecting CSV to auto-split → it won’t; use `List<T>` with repeated keys, or split yourself.
* Forgetting `defaultValue` makes it required → add it or use wrappers/`Optional`.
* Primitives for optional params → switch to wrappers.
* Enum case mismatches → add a `Converter<String, Enum>` or normalize input.
* Relying on empty string vs default → `defaultValue` fires for empty too.

---

## 18) Minimal, idiomatic examples

```java
// 1) Search with sane defaults
@GetMapping("/v1/search")
public List<Item> search(
  @RequestParam String q,
  @RequestParam(defaultValue = "1") int page,
  @RequestParam(defaultValue = "20") int size
) { /* ... */ }

// 2) Filters: multi-value categories
@GetMapping("/v1/items")
public List<Item> byCategory(@RequestParam List<String> category) { /* ... */ }

// 3) Optional toggle
@GetMapping("/v1/users")
public List<User> users(@RequestParam Optional<Boolean> active) { /* ... */ }

// 4) File + form fields
@PostMapping(value="/v1/avatar", consumes=MediaType.MULTIPART_FORM_DATA_VALUE)
public void avatar(@RequestParam MultipartFile file,
                   @RequestParam(required=false) String note) { /* ... */ }
```

---

## 19) Quick reference table

| Need                | Annotation & type                                           | Example               |
| ------------------- | ----------------------------------------------------------- | --------------------- |
| Required scalar     | `@RequestParam String`                                      | `?q=java`             |
| Optional scalar     | `@RequestParam(required=false) String` / `Optional<String>` | `?q=` or missing      |
| Defaulted scalar    | `@RequestParam(defaultValue="1") int`                       | missing → `1`         |
| Multi-value         | `@RequestParam List<T>` / `T[]`                             | `?tag=a&tag=b`        |
| Arbitrary bag       | `@RequestParam Map<String,String>`                          | any keys              |
| Preserve duplicates | `@RequestParam MultiValueMap<String,String>`                | `?k=1&k=2`            |
| File                | `@RequestParam MultipartFile`                               | `multipart/form-data` |
| Date/Time           | `@DateTimeFormat(...)` + type                               | ISO formats           |
| Custom type         | `Converter<String,T>`                                       | global converter      |

---

### Mental model

`@RequestParam` is **thin glue** between text in the URL or form and your Java types. Keep it small, explicit, and validated. When data gets complex, graduate to `@ModelAttribute` or `@RequestBody`.
