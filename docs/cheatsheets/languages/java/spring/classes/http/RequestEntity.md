---
title: RequestEntity<T>
date: 2025-10-21
tags: 
    - java
    - spring
    - http
    - request
summary: A cheatsheet for Spring's RequestEntity<T>, detailing its purpose, usage patterns, and examples for building and sending HTTP requests with full control over method, URI, headers, and body.
aliases:
  - Spring RequestEntity
---



# 📬 `RequestEntity<T>` — Spring’s HTTP Request Wrapper Cheatsheet

> **Essence:**
> `RequestEntity<T>` represents the *entire HTTP request* you’re about to send or that your controller received:
> **method + URI + headers + (optional) body `T`**.
> Use it for precise, testable, framework-friendly HTTP calls and for accessing raw request metadata in controllers.

---

## 1) Where it lives

* **Package:** `org.springframework.http`
* **Extends:** `HttpEntity<T>` (adds method + URI)
* Used by: **RestTemplate**, **WebClient adaptors**, and as a **controller parameter** to read inbound requests.

```java
import org.springframework.http.RequestEntity;
```

---

## 2) When to use it (and why)

| Situation                                                      | Problem without it             | Why `RequestEntity`                                       |
| -------------------------------------------------------------- | ------------------------------ | --------------------------------------------------------- |
| You need strict control over **method/URI/headers/body**       | Ad-hoc method params get messy | Single immutable object with everything                   |
| You want a **clean contract** for `RestTemplate.exchange(...)` | Overloaded method zoo          | `exchange(RequestEntity, Class)` is explicit and testable |
| Controller must **read headers + body + method**               | `@RequestBody` only gives body | Accept `RequestEntity<T>` in method args                  |
| You want **conditional / cache / auth** headers                | Easy to forget or misplace     | Chain them on the builder in one place                    |

---

## 3) Anatomy

```
RequestEntity<T>
├─ HttpMethod method
├─ URI url
├─ HttpHeaders headers
└─ T body (nullable)
```

Immutable value object; thread-safe to share.

---

## 4) Building requests (fluent API)

```java
URI uri = URI.create("https://api.example.com/users/42");

RequestEntity<Void> req = RequestEntity
    .get(uri)
    .accept(MediaType.APPLICATION_JSON)
    .header("X-Trace-Id", traceId)
    .build();

RequestEntity<UserCreateDto> post = RequestEntity
    .post(URI.create("https://api.example.com/users"))
    .contentType(MediaType.APPLICATION_JSON)
    .accept(MediaType.APPLICATION_JSON)
    .body(payload);
```

Available builders: `get`, `post`, `put`, `patch`, `delete`, `head`, `options`, `method(HttpMethod, URI)`.

---

## 5) Sending with RestTemplate

```java
RestTemplate rt = new RestTemplate();

// No body
RequestEntity<Void> getReq = RequestEntity.get(URI.create(url))
    .accept(MediaType.APPLICATION_JSON).build();
ResponseEntity<User> getRes = rt.exchange(getReq, User.class);

// With body
RequestEntity<UserCreateDto> postReq = RequestEntity.post(URI.create(url))
    .contentType(MediaType.APPLICATION_JSON)
    .body(dto);
ResponseEntity<User> postRes = rt.exchange(postReq, User.class);
```

**Generics:** For collections/maps, use `ParameterizedTypeReference`:

```java
RequestEntity<Void> req = RequestEntity.get(URI.create(url)).build();
ResponseEntity<List<User>> res = rt.exchange(req, new ParameterizedTypeReference<>() {});
List<User> users = res.getBody();
```

---

## 6) Using it in controllers (read raw request)

```java
@PostMapping("/ingest")
public ResponseEntity<Void> ingest(RequestEntity<String> request) {
    HttpMethod method   = request.getMethod();
    URI        uri      = request.getUrl();
    HttpHeaders headers = request.getHeaders();
    String     body     = request.getBody();
    // ... validate, route, log
    return ResponseEntity.accepted().build();
}
```

Works for MVC and WebFlux controller methods.

---

## 7) Common header patterns

```java
RequestEntity<Void> cond = RequestEntity.get(URI.create(url))
    .header(HttpHeaders.IF_NONE_MATCH, etag)
    .header(HttpHeaders.IF_MODIFIED_SINCE, httpDate)
    .build();

RequestEntity<Resource> upload = RequestEntity
    .post(URI.create(url))
    .contentType(MediaType.MULTIPART_FORM_DATA)
    .body(new ByteArrayResource(bytes));
```

Tip: for auth, prefer an interceptor, but you can inline:

```java
RequestEntity<Void> authed = RequestEntity.get(URI.create(url))
    .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
    .build();
```

---

## 8) Query params & path variables (compose the URI)

```java
URI uri = UriComponentsBuilder
    .fromHttpUrl("https://api.example.com/search")
    .queryParam("q", "spring+http")
    .queryParam("limit", 20)
    .build(true)     // keep encoded
    .toUri();

RequestEntity<Void> req = RequestEntity.get(uri).build();
```

---

## 9) Binary & streaming bodies

```java
RequestEntity<ByteArrayResource> bin = RequestEntity
    .post(URI.create(url))
    .contentType(MediaType.APPLICATION_OCTET_STREAM)
    .body(new ByteArrayResource(bytes));

RequestEntity<InputStreamResource> stream = RequestEntity
    .post(URI.create(url))
    .contentType(MediaType.APPLICATION_OCTET_STREAM)
    .body(new InputStreamResource(inputStream));
```

---

## 10) Multipart forms (file + JSON)

```java
MultiValueMap<String, Object> parts = new LinkedMultiValueMap<>();
parts.add("file", new ByteArrayResource(bytes){ @Override public String getFilename(){ return "data.csv"; }});
parts.add("meta", new HttpEntity<>(Map.of("source", "import"), new HttpHeaders(){{
    setContentType(MediaType.APPLICATION_JSON);
}}));

RequestEntity<MultiValueMap<String, Object>> req = RequestEntity
    .post(URI.create(url))
    .contentType(MediaType.MULTIPART_FORM_DATA)
    .body(parts);
```

---

## 11) Conditional requests & caching

```java
RequestEntity<Void> req = RequestEntity.get(URI.create(url))
    .ifNoneMatch(etag)             // shortcut for If-None-Match
    .build();
```

If the resource hasn’t changed, expect **304 Not Modified**.

---

## 12) Comparison with siblings

| Type                | Direction           | Includes                          | Typical use                      |
| ------------------- | ------------------- | --------------------------------- | -------------------------------- |
| `HttpEntity<T>`     | Request or Response | **headers + body**                | Simplest wrapper                 |
| `RequestEntity<T>`  | Request             | **method + URI + headers + body** | Outbound calls, controller input |
| `ResponseEntity<T>` | Response            | **status + headers + body**       | Controller output, client result |

---

## 13) Testing with MockRestServiceServer (client-side)

```java
MockRestServiceServer server = MockRestServiceServer.bindTo(rt).build();

server.expect(requestTo("https://api.example.com/users/42"))
      .andExpect(method(HttpMethod.GET))
      .andExpect(header("X-Trace-Id", traceId))
      .andRespond(withSuccess("""{"id":42}""", MediaType.APPLICATION_JSON));

ResponseEntity<User> res = rt.exchange(
    RequestEntity.get(URI.create("https://api.example.com/users/42"))
                 .header("X-Trace-Id", traceId)
                 .build(),
    User.class
);
```

---

## 14) Pitfalls (and fixes)

| Mistake                                                              | Symptom                    | Fix                                                                                           |
| -------------------------------------------------------------------- | -------------------------- | --------------------------------------------------------------------------------------------- |
| Building `RequestEntity` but calling the wrong `RestTemplate` method | Headers/method ignored     | Use `exchange(RequestEntity, Class)` or `exchange(RequestEntity, ParameterizedTypeReference)` |
| Forgetting `contentType` on POST/PUT                                 | 415 Unsupported Media Type | Set `.contentType(MediaType.APPLICATION_JSON)` (or correct type)                              |
| Manually concatenating query params                                  | Encoding bugs              | Use `UriComponentsBuilder`                                                                    |
| Leaking auth tokens in logs                                          | Security risk              | Mask/redact headers in logging interceptors                                                   |
| Using `RequestEntity` with WebClient as-is                           | Awkward API                | Prefer WebClient’s fluent style; convert pieces if needed                                     |

---

## 15) Quick reference

```java
// GET no body
RequestEntity<Void> r1 = RequestEntity.get(URI.create(u))
    .accept(MediaType.APPLICATION_JSON).build();

// POST JSON body
RequestEntity<MyDto> r2 = RequestEntity.post(URI.create(u))
    .contentType(MediaType.APPLICATION_JSON).body(dto);

// Custom method + headers
RequestEntity<Void> r3 = RequestEntity.method(HttpMethod.HEAD, URI.create(u))
    .header("X-Foo", "bar").build();

// Controller: read inbound request
ResponseEntity<Void> handle(RequestEntity<String> req) { ... }
```

---

## 16) Mental model

```
[RequestEntity<T>] → one immutable package of:
  method + URI + headers + body
        ↓
RestTemplate.exchange(...)
        ↓
[ResponseEntity<R>]
```

Design once, test easily, send consistently. This keeps HTTP concerns explicit and localized—cleaner than a dozen ad-hoc parameters sprinkled through your code.
