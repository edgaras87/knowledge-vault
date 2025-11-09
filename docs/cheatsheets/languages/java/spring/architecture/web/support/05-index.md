---
title: Content
date: 2025-11-08
tags: 
    - java
    - spring
    - architecture
    - web
    - support
    - cheatsheet
summary: A comprehensive overview of the Web Support layer in a Java Spring application, detailing its structure, purpose, and best practices for managing HTTP interactions, URI building, error handling, and mapping within the web edge of the application.
aliases:
    - Java Spring Web Support - Content Cheatsheet
---



# Web Support — Index

The web edge is where raw HTTP meets your application.  
This area collects tiny, composable helpers that shape **URIs**, **errors**, **HTTP headers**, and **web-side mapping**.  
They keep controllers small, keep policies centralized, and keep the boundary sharp.

This section is organized into **four subpackages**, each with a clear responsibility:

```
web/support/
links/        # URI & hyperlink policy
errors/       # RFC7807 ProblemDetail vocabulary
http/         # HTTP headers, caching, content-disposition extras
mapping/      # Mapper contexts & qualifiers
```

Feature-specific support still lives under:

```
web/<feature>/support/
```

Cross-feature, API-wide rules live here.

---

## Why this folder exists

- A stable web boundary needs **small, reusable policies**.
- Controllers shouldn’t build URLs by hand or assemble ProblemDetail.
- Mappers shouldn’t know where the API base is.
- Error responses shouldn’t drift across endpoints.
- Pagination, ETags, and Content-Disposition need one consistent home.

This folder is the **diplomatic interface** between HTTP and the rest of your application.

---

## Subpackages Overview

### `links/` — URI & Link Builders

Everything that composes **public URIs** and **navigation links**.

**Typical inhabitants**

- `UriFactory` — normalize `app.host` once; build bases (`/api/`, `/problems/`)
- `ApiLinks` — shared `/api/` base resolver
- `ResourceLinks` (optional shared version; feature-scoped versions live in `web/<feature>/support/`)
- `PaginationLinks` — `self/first/prev/next/last`
- `LinkHeader` — builds the HTTP `Link:` header

**Purpose**

Keep URL math out of controllers. Centralize link shape so clients get predictable navigation.

---

### `errors/` — ProblemDetail & Error Vocabulary

Everything for **RFC-7807**, slugs, titles, and translation.

**Typical inhabitants**

- `ProblemCatalog` — stable slugs for all problems
- `ProblemLinks` — `type(slug)` → `https://…/problems/{slug}`
- `ProblemText` — titles via `MessageSource` (`problems.<slug>.title`)
- `ProblemFactory` — tiny builder for ProblemDetail (slug + status + detail)
- `ExceptionMappings` — machine metadata per slug (`status`, `code`)
- `GlobalExceptionHandler` — translates exceptions → ProblemDetail

**Purpose**  

Give your API a stable **error language**. Identity (slugs) in code; presentation (titles) in bundles.

---

### `http/` — HTTP Header & Response Policy

Everything that shapes headers or download responses.

**Typical inhabitants**

- `CacheControlPolicy` — consistent cache rules (no-store, short-lived, immutable)
- `EtagFactory` — weak/strong ETags from stable representations
- `ContentDispositionSafe` — safe UTF-8 filenames with ASCII fallback
- `CorrelationIds` (optional) — attach request IDs to problems or logs

**Purpose**

Attach headers safely, consistently, and without repeating boilerplate.

---

### `mapping/` — Web-Side Mapping Helpers

Helpers that enrich MapStruct or manual mappers with web-edge context.

**Typical inhabitants**

- `WebMappingContext` — exposes `ApiLinks`, `ProblemLinks`, locale
- Qualifiers (e.g., `@ToSelfLink`) to keep mapping expressions clean
- Lightweight formatters (e.g., `DateTimeFormatters`)
- *(Note: feature mappers do **not** live here — only shared mapping inputs.)*

**Purpose**

Let mappers produce DTOs with correct links, locales, and formats without contaminating mapping code with HTTP logic.

---

## Relationship to feature support

```
web/<feature>/support/
UserResourceLinks.java
UserQueryParser.java
UserExceptionTranslator.java
```

Feature support is **local** to a feature.  

Shared support is **global** API vocabulary.

**Rule of thumb:**

- If its behavior affects multiple controllers → `web/support/`  
- If only one feature needs it → `web/<feature>/support/`

---

## Folder Structure (Reference)

```text
src/main/java/com/example/app/web/
└─ support/
   ├─ links/
   │  ├─ UriFactory.java
   │  ├─ ApiLinks.java
   │  ├─ PaginationLinks.java
   │  └─ LinkHeader.java
   ├─ errors/
   │  ├─ ProblemCatalog.java
   │  ├─ ProblemLinks.java
   │  ├─ ProblemText.java
   │  ├─ ProblemFactory.java
   │  ├─ ExceptionMappings.java
   │  └─ GlobalExceptionHandler.java
   ├─ http/
   │  ├─ CacheControlPolicy.java
   │  ├─ EtagFactory.java
   │  └─ ContentDispositionSafe.java
   └─ mapping/
      ├─ WebMappingContext.java
      ├─ ToSelfLink.java          # example qualifier
      └─ DateTimeFormatters.java          
```

---

## What does *not* belong here

* Domain rules
* Application use-case logic
* Repositories & JPA entities
* Global MapStruct config (`infra/mapping/MapStructConfig`)
* Feature-specific DTOs or mappers

This folder is **purely HTTP-facing**.

---

## How controllers benefit

Controllers become tiny:

```java
return ResponseEntity.created(userLinks.location(id)).build();
```

Error responses are uniform:

```java
throw new UserNotFound(id); // translated once in GlobalExceptionHandler
```

---

## Sanity Checklist

* Shared HTTP rules → `web/support/`
* Feature-specific rules → `web/<feature>/support/`
* No business logic or persistence calls inside support
* Only stateless, constructor-injected helpers
* No URL string math in controllers
* Slugs in code; titles in resource bundles
* Tests cover URI building & ProblemDetail shape


