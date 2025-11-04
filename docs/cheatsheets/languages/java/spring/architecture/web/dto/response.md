---
title: Response
date: 2025-11-04
tags: 
    - dto
    - response
    - rest
    - spring-boot
    - design
summary: How we shape API responses - naming, structure, envelopes, expansions, and conventions.
aliases:
  - Response DTOs Cheatsheet
---

# `web/user/dto/response` — Cheatsheet (shapes, rules, templates)

This is your one-stop, copy-pasteable playbook for **outbound payloads** from the `web` edge. It defines what we return, why, and how to keep responses boringly consistent as the project grows.

---

## 0) Purpose & boundaries

* **Goal:** Stable, explicit **response shapes** for the HTTP layer.
* **Lives here:** `src/main/java/com/example/app/web/user/dto/response/**`
* **Owns:** Field names, flattening policy, envelopes (single/collection/page), JSON conventions, mapping targets.
* **Does not own:** Persistence, business rules, transactionality (those live in `app/` and `domain/`).

Design mantra: **Request → Command → Entity/Projection → Response**
Responses are immutable value types (Java `record` by default).

---

## 1) Directory layout

```
web/user/dto/response/
├─ UserListItemResponse.java
├─ UserDetailResponse.java
├─ UserSummaryResponse.java
└─ envelopes/
   ├─ SingleEnvelope.java
   ├─ CollectionEnvelope.java
   └─ PageEnvelope.java
```

> Naming: **`*Response` only** for outputs (no `Dto` suffix). Keep it per feature/module.

---

## 2) JSON conventions (project-wide)

* **Case:** `snake_case` or `camelCase` — pick one and stick to it. Examples below use `camelCase`.
* **Null policy:** Prefer **omitting** truly optional/unknown fields → `@JsonInclude(NON_NULL)`.
* **IDs:** String or long? Choose once (often `Long` internally, `String` externally if you’ll switch to ULIDs later). This sheet uses `String`.
* **Timestamps:** Always **UTC ISO-8601** strings (`Instant`) at the edge.
* **Booleans:** Never tri-state. If “unknown”, omit the field.
* **Enums:** Serialize as **lowercase strings** with a stable vocabulary.
* **Numbers:** No business formatting (currency symbols) in the API — that’s a client concern.

---

## 3) Flattening policy (future-proof sanity)

* **Default:** **Flatten one level** for core properties used 80% of the time (denormalize the obvious).
* **Nested objects:** Keep only when the nested concept is meaningful on its own (e.g., `profile`, `address`).
* **Collections:** Summaries in list items (count, top 3), **full collections in detail only** when needed.
* **Rule of thumb:** Lists show **“identify + preview”**; details show **“complete but bounded”**.

---

## 4) Core response records (copy-paste)

### `UserListItemResponse`

For tables and autocomplete lists. Small, stable shape.

```java
package com.example.app.web.user.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonPropertyOrder;

import java.time.Instant;

@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonPropertyOrder({ "id", "email", "name", "status", "createdAt" })
public record UserListItemResponse(
    String id,
    String email,
    String name,
    String status,      // e.g., "active", "disabled"
    Instant createdAt   // always UTC; edge type is Instant
) {}
```

**Example JSON**

```json
{
  "id": "u_90421",
  "email": "mira@example.com",
  "name": "Mira Lane",
  "status": "active",
  "createdAt": "2025-10-31T12:03:15Z"
}
```

---

### `UserDetailResponse`

For `GET /users/{id}` — includes richer fields, optional nested blocks.

```java
package com.example.app.web.user.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonPropertyOrder;
import java.time.Instant;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonPropertyOrder({
  "id", "email", "name", "status",
  "profile", "roles",
  "createdAt", "updatedAt"
})
public record UserDetailResponse(
    String id,
    String email,
    String name,
    String status,
    Profile profile,
    List<String> roles,
    Instant createdAt,
    Instant updatedAt
) {
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public record Profile(
      String displayName,
      String avatarUrl,
      String timezone
  ) {}
}
```

**Example JSON**

```json
{
  "id": "u_90421",
  "email": "mira@example.com",
  "name": "Mira Lane",
  "status": "active",
  "profile": {
    "displayName": "Mira",
    "avatarUrl": "https://cdn.example.com/avatars/u_90421.png",
    "timezone": "Europe/Vilnius"
  },
  "roles": ["user", "admin"],
  "createdAt": "2025-10-31T12:03:15Z",
  "updatedAt": "2025-11-01T09:02:11Z"
}
```

---

### `UserSummaryResponse`

For sidebars, audit logs, or embedding in other aggregates.

```java
package com.example.app.web.user.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record UserSummaryResponse(
    String id,
    String email,
    String name
) {}
```

---

## 5) Envelopes (consistent top-level shapes)

### When to use envelopes

* **SingleEnvelope** – when you might add `links`, `meta`, or `included` later.
* **CollectionEnvelope** – for non-paged lists; always include a `count`.
* **PageEnvelope** – for server-side pagination (stable contract for FE).

> All envelopes are **generic** and reusable across features.

### `SingleEnvelope<T>`

```java
package com.example.app.web.user.dto.response.envelopes;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record SingleEnvelope<T>(
    T data,
    Meta meta,
    Links links
) {
  public record Meta(String version) {}
  public record Links(String self) {}
}
```

**JSON**

```json
{
  "data": { "id": "u_90421", "email": "mira@example.com", "name": "Mira Lane" },
  "meta": { "version": "1" },
  "links": { "self": "/api/users/u_90421" }
}
```

---

### `CollectionEnvelope<T>`

```java
package com.example.app.web.user.dto.response.envelopes;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record CollectionEnvelope<T>(
    List<T> data,
    Integer count,
    Links links
) {
  public record Links(String self) {}
}
```

**JSON**

```json
{
  "data": [
    { "id": "u_1", "email": "a@x.com", "name": "A" },
    { "id": "u_2", "email": "b@x.com", "name": "B" }
  ],
  "count": 2,
  "links": { "self": "/api/users?status=active" }
}
```

---

### `PageEnvelope<T>`

Minimal, predictable, front-end friendly.

```java
package com.example.app.web.user.dto.response.envelopes;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record PageEnvelope<T>(
    List<T> data,
    Page page,
    Links links
) {
  public record Page(
      int number,        // 0-based
      int size,          // requested size
      long totalElements,
      int totalPages
  ) {}
  public record Links(String self, String next, String prev) {}
}
```

**JSON**

```json
{
  "data": [
    { "id": "u_1", "email": "a@x.com", "name": "A" }
  ],
  "page": { "number": 0, "size": 20, "totalElements": 41, "totalPages": 3 },
  "links": { "self": "/api/users?page=0&size=20", "next": "/api/users?page=1&size=20" }
}
```

---

## 6) Mapping strategies (Entity/Projection → Response)

**Prefer MapStruct** for compile-time mapping, explicit and fast.

```java
package com.example.app.web.user.mapper;

import com.example.app.domain.user.User;
import com.example.app.web.user.dto.response.*;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface UserOutputMapper {

  @Mapping(target = "status", source = "status.value") // if domain wraps it
  UserListItemResponse toListItem(User user);

  @Mapping(target = "profile.displayName", source = "profile.displayName")
  @Mapping(target = "profile.avatarUrl", source = "profile.avatarUrl")
  @Mapping(target = "profile.timezone", source = "profile.timezone")
  UserDetailResponse toDetail(User user);

  UserSummaryResponse toSummary(User user);

  // Overloads for projections (avoid N+1):
  UserListItemResponse toListItem(UserListProjection p);
  UserDetailResponse toDetail(UserDetailProjection p);
}
```

**Projections** (in `infra/user/`) let you fetch exactly what the response needs:

```java
public interface UserListProjection {
  String getId();
  String getEmail();
  String getName();
  String getStatus();
  java.time.Instant getCreatedAt();
}
```

> Rule: **List endpoints map from projections**; detail endpoints can map from entity or dedicated detail projection.

---

## 7) Versioning & compatibility

* Start with **URL** or **media type** versioning (pick one).
  Example: `Accept: application/vnd.example.user+json;version=1`
* When adding fields, **only add optional fields** or defaults; never silently change semantics.
* When you must break, add `v2` responses in parallel (new records & controller methods).

---

## 8) Error representation (heads-up)

Even though errors are handled in `problem/`, your **success envelopes** must stay stable **alongside** the Problem Details contract. Keep both predictable. Don’t mix success `data` with error fields.

---

## 9) Testing JSON shape (micro-templates)

### Controller slice — list endpoint (shape + envelope)

```java
// src/test/java/com/example/app/web/user/UserControllerTest.java
@AutoConfigureMockMvc
@SpringBootTest
class UserControllerTest {

  @Autowired MockMvc mvc;

  @Test
  void listUsers_returnsPageEnvelopeOfListItems() throws Exception {
    mvc.perform(get("/api/users").header("Authorization", "Bearer demo"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.data").isArray())
       .andExpect(jsonPath("$.data[0].id").exists())
       .andExpect(jsonPath("$.page.number").value(0))
       .andExpect(jsonPath("$.links.self").exists());
  }
}
```

### JSON contract unit (no Spring) — serialize one DTO

```java
var dto = new UserListItemResponse("u_1","a@x.com","A","active", Instant.parse("2025-11-01T00:00:00Z"));
var json = new ObjectMapper().writeValueAsString(dto);
assertThat(json).contains("\"email\":\"a@x.com\"");
```

---

## 10) Common pitfalls & how to dodge them

* **Leaking internals:** Never expose JPA entities or domain types directly.
* **Inconsistent paging:** Always wrap list endpoints in `CollectionEnvelope` or `PageEnvelope`. Don’t free-hand arrays sometimes.
* **Surprise nulls:** Omit unknown/irrelevant fields; don’t write `null` unless it conveys meaning.
* **Configurable naming:** Don’t let Jackson global config drift; set a project standard and annotate DTOs if you must override.
* **Timestamp chaos:** Convert to `Instant` at the edge; store/compute however you like inside.

---

## 11) Quick wiring examples (controller returns)

**Get by id (single envelope)**

```java
@GetMapping("/{id}")
public ResponseEntity<SingleEnvelope<UserDetailResponse>> get(@PathVariable String id) {
  var user = service.get(id); // domain entity or projection
  var body = new SingleEnvelope<>(
      outputMapper.toDetail(user),
      new SingleEnvelope.Meta("1"),
      new SingleEnvelope.Links("/api/users/" + id)
  );
  return ResponseEntity.ok(body);
}
```

**List (paged envelope)**

```java
@GetMapping
public ResponseEntity<PageEnvelope<UserListItemResponse>> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size
) {
  var slice = service.search(page, size); // returns page result + totals
  var items = slice.items().stream().map(outputMapper::toListItem).toList();

  var body = new PageEnvelope<>(
      items,
      new PageEnvelope.Page(page, size, slice.totalElements(), slice.totalPages()),
      new PageEnvelope.Links("/api/users?page=%d&size=%d".formatted(page, size),
                             slice.hasNext() ? "/api/users?page=%d&size=%d".formatted(page+1, size) : null,
                             page > 0 ? "/api/users?page=%d&size=%d".formatted(page-1, size) : null)
  );
  return ResponseEntity.ok(body);
}
```

---

## 12) Field selection matrix (cheat)

| Context          | ID | Email | Name | Status | Roles | Profile | Created | Updated |
| ---------------- | -- | ----- | ---- | ------ | ----- | ------- | ------- | ------- |
| ListItemResponse | ✔︎ | ✔︎    | ✔︎   | ✔︎     | –     | –       | ✔︎      | –       |
| DetailResponse   | ✔︎ | ✔︎    | ✔︎   | ✔︎     | ✔︎    | ✔︎      | ✔︎      | ✔︎      |
| SummaryResponse  | ✔︎ | ✔︎    | ✔︎   | –      | –     | –       | –       | –       |

Keep it boring and predictable.

---

## 13) Evolving safely

* New field? Add to **detail** first; promote to list item only when it’s truly ubiquitous.
* New nested block? Introduce as optional and document default behavior.
* Big refactor? Version the response (add `v2` types) and run both for a while.

---

## 14) TL;DR blueprint

* Records + `@JsonInclude(NON_NULL)`
* Three responses: **ListItem**, **Detail**, **Summary**
* Three envelopes: **Single**, **Collection**, **Page**
* Map from **projections** for lists, **entity or detail projection** for detail
* UTC `Instant`, stable IDs, enums as lowercase strings
* Tests assert **envelope + core fields**, not implementation trivia



## Extra resources

It’s solid for a beginner. You can ship with this and not look silly.
If you want to round off the sharp edges without bloating it, bolt on these tiny, high-leverage extras:

---

### A) One-time Jackson policy (project-wide)

Make DTO behavior predictable everywhere.

```java
// src/main/java/com/example/app/config/JacksonConfig.java
@Configuration
public class JacksonConfig {
  @Bean
  ObjectMapper objectMapper(Jackson2ObjectMapperBuilder b) {
    return b
        .serializationInclusion(JsonInclude.Include.NON_NULL)
        .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .build();
  }
}
```

* Dates as ISO-8601 (not timestamps).
* Omit nulls globally so you don’t sprinkle `@JsonInclude` everywhere.

---

### B) Minimal OpenAPI annotations on responses (discoverability)

Just enough for good docs; not noisy.

```java
@Operation(summary = "Get user by id")
@ApiResponses({
  @ApiResponse(responseCode = "200", description = "User found",
    content = @Content(mediaType = "application/json",
      schema = @Schema(implementation = SingleEnvelope.class))),
  @ApiResponse(responseCode = "404", description = "Not found")
})
@GetMapping("/{id}")
public ResponseEntity<SingleEnvelope<UserDetailResponse>> get(@PathVariable String id) { ... }
```

This makes your `SingleEnvelope<UserDetailResponse>` show up nicely in Swagger UI.

---

### C) Snapshot (“golden file”) tests for DTO shapes

Cheap insurance against accidental contract drift.

```java
// src/test/java/com/example/app/web/user/dto/response/UserListItemResponseJsonTest.java
class UserListItemResponseJsonTest {
  @Test
  void snapshot() throws Exception {
    var dto = new UserListItemResponse("u_1","a@x.com","A","active",
                                       Instant.parse("2025-11-01T00:00:00Z"));
    var json = new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .setSerializationInclusion(JsonInclude.Include.NON_NULL)
        .writeValueAsString(dto);

    // compare to stored test resource (src/test/resources/__snapshots__/UserListItemResponse.json)
    var expected = Files.readString(Path.of("src/test/resources/__snapshots__/UserListItemResponse.json"));
    assertThat(json).isEqualToIgnoringWhitespace(expected);
  }
}
```

Store one pretty-printed JSON per DTO. If something changes, tests shout.

---

### D) Sorting & filtering contract (document once)

Clients need to know the rules.

* **Filtering:** `GET /api/users?status=active&email=like:example.com`
* **Sorting:** `sort=createdAt,desc` (comma for direction; multiple `sort` params allowed)
* **Paging:** `page=0&size=20`, capped with `size<=100`
* Echo these in `PageEnvelope.links.self`.

Add a tiny table to the cheatsheet:

| Param  | Type  | Example            | Notes             |
| ------ | ----- | ------------------ | ----------------- |
| status | enum  | `active`           | lowercase strings |
| email  | like  | `like:example.com` | `%` implied       |
| sort   | field | `createdAt,desc`   | multiple allowed  |
| page   | int   | `0`                | 0-based           |
| size   | int   | `20`               | max 100           |

---

### E) Redaction & exposure checklist (save future pain)

Before exposing new fields, ask:

* Contains secrets/PII? If yes, **don’t return** (or return masked, e.g., `email`: `mi***@example.com` in list).
* Could it be gamed? (e.g., internal flags, scores) → do not expose.
* Is it stable? If not, hide behind a **detail-only** field.

---

### F) Backward-compatible evolution policy (one paragraph)

* Additive changes only (new optional fields) in v1.
* Breaking changes → new `v2` response types and endpoints; run v1 and v2 in parallel for a deprecation window.
* Document deprecations in `meta` or response headers (`Deprecation`, `Sunset`).

---

### G) Caching headers for read endpoints (tiny but mighty)

DTOs stay the same, but responses become faster and cheaper.

* **ETag**: hash of JSON body; return `304 Not Modified` on `If-None-Match`.
* **Cache-Control**: `max-age=30, must-revalidate` for list endpoints if data is hot.

You don’t change DTO code—only controller/filters—but it meaningfully upgrades your API.

---

### H) “Big list” escape hatch

When a list endpoint could return thousands, protect yourself:

* Hard cap `size` (e.g., 100).
* For truly large exports, use **asynchronous job + download link** pattern.
* Keep `PageEnvelope` for everything else.

---

### I) Tiny MapStruct affordances (quality-of-life)

Centralize time/ID translations so DTOs stay clean.

```java
@Mapper(componentModel = "spring", uses = { TimeMapper.class, IdMapper.class })
public interface UserOutputMapper { ... }

@Component
public class TimeMapper {
  public Instant toInstant(OffsetDateTime odt) { return odt == null ? null : odt.toInstant(); }
}

@Component
public class IdMapper {
  public String toPublicId(Long id) { return id == null ? null : "u_" + id; }
}
```

---

### J) Error shape parity reminder

Success uses envelopes; errors use **Problem Details** (RFC 7807). Never mix.
That line in the sand keeps clients sane.

---

If you add A, C, and E, you’re above beginner and already “junior-pro”: predictable JSON, contract tests, and a safety checklist. The rest are incremental upgrades when you feel the itch.
