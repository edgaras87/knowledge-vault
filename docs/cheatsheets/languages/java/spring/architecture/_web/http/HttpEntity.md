---
title: HttpEntity<T>
date: 2025-10-21
tags: 
    - java
    - spring
    - http
    - entity
summary: A cheatsheet for Spring's HttpEntity<T>, detailing its purpose, usage patterns, and examples for building HTTP requests and responses with headers and body.
aliases:
  - Spring HttpEntity
---

# 📦 `HttpEntity<T>` — Headers + Body, No Extras

> **Essence:**
> `HttpEntity<T>` is the **minimal HTTP envelope**: **headers + body**.
> No status, no method, no URI. It’s the base building block used by `RequestEntity` (adds method+URI) and `ResponseEntity` (adds status).

---

## 1) Where it lives

* **Package:** `org.springframework.http`
* **Role:** Base class for HTTP message content (request **or** response)
* **Used by:** Spring MVC & WebFlux, `RestTemplate`, message converters

```java
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
```

---

## 2) Anatomy

```
HttpEntity<T>
├─ HttpHeaders headers
└─ T body (nullable)
```

Immutable once created.

---

## 3) Why/when to use it

| Situation                                                                | Use `HttpEntity<T>` because…          |
| ------------------------------------------------------------------------ | ------------------------------------- |
| You only need to pass **headers + body** (no method/URI/status)          | It’s the smallest correct abstraction |
| You’re inside a controller and want access to inbound **headers + body** | Accept `HttpEntity<T>` as a parameter |
| You’re making a client call where some APIs accept `HttpEntity`          | Clean way to attach headers to a body |

If you need **method+URI**, use **`RequestEntity<T>`**.
If you need **status**, use **`ResponseEntity<T>`**.

---

## 4) Create it

```java
var headers = new HttpHeaders();
headers.set("X-Trace-Id", traceId);
headers.setContentType(MediaType.APPLICATION_JSON);

HttpEntity<MyDto> entity = new HttpEntity<>(payload, headers);
```

No builder; it’s just a value object.

---

## 5) In controllers: read raw request

```java
@PostMapping("/ingest")
public ResponseEntity<Void> ingest(HttpEntity<String> request) {
    HttpHeaders h = request.getHeaders();
    String body   = request.getBody();
    // validate, log, route...
    return ResponseEntity.accepted().build();
}
```

Works in MVC and WebFlux controllers.

---

## 6) With `RestTemplate`

```java
RestTemplate rt = new RestTemplate();

// POST JSON with headers
HttpHeaders h = new HttpHeaders();
h.setContentType(MediaType.APPLICATION_JSON);
HttpEntity<UserCreate> req = new HttpEntity<>(dto, h);

ResponseEntity<User> res = rt.postForEntity(url, req, User.class);
```

For GET with headers but no body:

```java
HttpHeaders h = new HttpHeaders();
h.setAccept(List.of(MediaType.APPLICATION_JSON));
HttpEntity<Void> req = new HttpEntity<>(h);

ResponseEntity<User> res = rt.exchange(url, HttpMethod.GET, req, User.class);
```

---

## 7) Generics: keep types honest

```java
HttpEntity<List<User>> entity = new HttpEntity<>(users, headers);
// Body type drives message converter selection (JSON, XML, etc.)
```

For responses with parameterized types, you still need `ParameterizedTypeReference` at the **ResponseEntity** side (client read path), not here.

---

## 8) Content negotiation & converters

* Spring picks a **`HttpMessageConverter`** based on:

  * The body type `T`
  * `Content-Type` (outbound) / `Accept` (inbound)
* `HttpEntity` just carries the data; converters do the serialization/deserialization.

---

## 9) Compared with siblings

| Type                | Direction           | Adds on top of `HttpEntity` | Typical use                                         |
| ------------------- | ------------------- | --------------------------- | --------------------------------------------------- |
| `HttpEntity<T>`     | Request or Response | —                           | Minimal headers+body wrapper                        |
| `RequestEntity<T>`  | Request             | **method + URI**            | Outbound requests, controller param with method/URI |
| `ResponseEntity<T>` | Response            | **status**                  | Controller return value, client result              |

---

## 10) Common patterns

**Upload binary**

```java
HttpHeaders h = new HttpHeaders();
h.setContentType(MediaType.APPLICATION_OCTET_STREAM);
HttpEntity<byte[]> req = new HttpEntity<>(bytes, h);
rt.exchange(url, HttpMethod.POST, req, Void.class);
```

**Multipart (file + JSON)**

```java
MultiValueMap<String,Object> parts = new LinkedMultiValueMap<>();
parts.add("file", new ByteArrayResource(bytes){ @Override public String getFilename(){ return "data.csv"; }});
parts.add("meta", new HttpEntity<>(Map.of("source","import"),
  new HttpHeaders(){ { setContentType(MediaType.APPLICATION_JSON); } }));

HttpHeaders h = new HttpHeaders();
h.setContentType(MediaType.MULTIPART_FORM_DATA);
HttpEntity<MultiValueMap<String,Object>> req = new HttpEntity<>(parts, h);
rt.postForEntity(url, req, Void.class);
```

**Conditional GET**

```java
HttpHeaders h = new HttpHeaders();
h.setIfNoneMatch(etag);
HttpEntity<Void> req = new HttpEntity<>(h);
ResponseEntity<Resource> res = rt.exchange(url, HttpMethod.GET, req, Resource.class);
// 304 if not modified
```

---

## 11) Pitfalls (don’t do these)

| Mistake                                                           | Why it bites               | Do instead                            |
| ----------------------------------------------------------------- | -------------------------- | ------------------------------------- |
| Returning `HttpEntity` from a controller expecting status control | You can’t set status here  | Return `ResponseEntity<T>`            |
| Trying to set method/URI on `HttpEntity`                          | It doesn’t have them       | Use `RequestEntity<T>`                |
| Forgetting `Content-Type` on POST/PUT                             | 415 Unsupported Media Type | `headers.setContentType(...)`         |
| Mutating headers after sending                                    | Too late / ignored         | Build headers before creating/sending |

---

## 12) Quick reference

```java
// Make one
HttpEntity<T> e1 = new HttpEntity<>(body);
HttpEntity<T> e2 = new HttpEntity<>(body, headers);
HttpEntity<Void> e3 = new HttpEntity<>(headers);

// Controller reads inbound
ResponseEntity<?> handle(HttpEntity<String> req) { ... }

// RestTemplate call with headers
rt.exchange(url, HttpMethod.GET, new HttpEntity<>(headers), Resp.class);
rt.postForEntity(url, new HttpEntity<>(dto, headers), Resp.class);
```

---

## 13) Mental model

```
[HttpEntity<T>]
  headers + body
     ↓
(used by converters)
     ↓
wire format (JSON/XML/Binary)
```

It’s the **plain envelope**. Reach for `RequestEntity` when you need route/method; reach for `ResponseEntity` when you need status semantics. Keep your API surface honest and your HTTP concerns explicit.
