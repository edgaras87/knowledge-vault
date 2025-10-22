---
title: "Request Binding Overview"
date: 2025-10-22
tags:
  - spring
  - java
  - web
  - mvc
summary: One mental model for all data-binding annotations in Spring MVC — understand when to use @PathVariable, @RequestParam, or @RequestBody, and how each maps incoming HTTP data into Java objects.
aliases:
  - Spring Web Binding Overview
---

# 🌐 Request Binding in Spring MVC

When an HTTP request hits your controller, Spring must **bind the incoming data** — path, query, form, or JSON body — to your method parameters.

There are three main “doors” for that data:

| Layer | Annotation | Source | Typical Example |
|-------|-------------|---------|----------------|
| **Identity** | [`@PathVariable`](path-variable.md) | Path segment | `/users/{id}` |
| **Filters / form inputs** | [`@RequestParam`](request-param.md) | Query string or form field | `/users?active=true&page=2` |
| **Payload** | [`@RequestBody`](request-body.md) | Request body (JSON/XML) | `{ "name": "Ana" }` |

---

## 🧠 Mental Model — *“Where does the data live?”*

```

┌──────────────────────────────┐
│       HTTP REQUEST           │
│                              │
│  GET /users/42?active=true   │
│  Body: { "name": "Ana" }     │
└─────────────┬────────────────┘
              │
              ▼
┌──────────────────────────────┐
│ Controller Method            │
│                              │
│ @PathVariable Long id        │ ← /users/42
│ @RequestParam Boolean active │ ← ?active=true
│ @RequestBody UserDto body    │ ← JSON body
└──────────────────────────────┘

```

---

## ⚙️ Core Purposes

| Intent | Use | Example |
|--------|-----|----------|
| **Identify a resource** | `@PathVariable` | `/users/{id}` |
| **Filter, sort, paginate** | `@RequestParam` | `/users?active=true&page=2` |
| **Create or update data** | `@RequestBody` | POST `/users` with `{...}` |

---

## 🧩 Engine Differences

| Annotation | Bound from | Conversion Engine | Typical Content-Type |
|-------------|-------------|-------------------|-----------------------|
| `@PathVariable` | URL template | `ConversionService` | `text/plain` (implicit) |
| `@RequestParam` | Query/form field | `ConversionService` | `application/x-www-form-urlencoded`, `multipart/form-data` |
| `@RequestBody` | Request body | `HttpMessageConverter` (e.g., Jackson JSON) | `application/json`, `application/xml` |

---

## ⚖️ Comparison Summary

| Dimension | `@PathVariable` | `@RequestParam` | `@RequestBody` |
|------------|----------------|----------------|----------------|
| **Source** | URL segment | Query string / form | HTTP body |
| **When used** | Identify which resource | Filter or adjust request | Send or receive structured data |
| **Default scope** | Required | Required (can default/optional) | Required |
| **Type conversion** | String → simple type | String → simple type | JSON/XML → object |
| **Cache key relevance** | Yes (different URL = different resource) | Yes (different query = different resource) | No (body not used in cache key) |
| **Validation level** | Simple | Simple | Deep (object graph) |
| **Good for** | REST paths | Filters, toggles, pagination | DTOs for create/update |
| **Don’t use for** | JSON bodies | Complex objects | Filters or identifiers |

---

## 🔍 Real-World Mapping Example

```
POST /shops/7/orders?express=true
Body: { "item": "Book", "qty": 3 }
```

```java
@PostMapping("/shops/{shopId}/orders")
public OrderDto createOrder(
    @PathVariable long shopId,           // /shops/7
    @RequestParam(defaultValue="false") boolean express, // ?express=true
    @RequestBody OrderCreateRequest body // { "item": "Book", "qty": 3 }
) { ... }
```

---

## 🧭 Folder Placement

```
cheatsheets/frameworks/spring/web/
└─ binding/
   ├─ overview.md          # ← this file
   ├─ path-variable.md
   ├─ request-param.md
   └─ request-body.md
```

In MkDocs navigation:

```yaml
- Spring Web:
  - Binding (overview): cheatsheets/frameworks/spring/web/binding/overview.md
  - PathVariable: cheatsheets/frameworks/spring/web/binding/path-variable.md
  - RequestParam: cheatsheets/frameworks/spring/web/binding/request-param.md
  - RequestBody: cheatsheets/frameworks/spring/web/binding/request-body.md
```

---

## 🪜 Next Layers

Once you master these three, your next frontier:

* [`@ModelAttribute`](../model-attribute.md) — group multiple `@RequestParam`s into a validated object.
* [`@ResponseBody`](../response-body.md) — the reverse: converting your method return value to JSON/XML.
* [`@MatrixVariable`](../matrix-variable.md) — niche but powerful for REST-like filtering in path segments.

---

**Bottom line:**

> `@PathVariable` locates the thing,
> `@RequestParam` describes *how* to get it,
> `@RequestBody` tells *what* it is.

Use this trinity as your mental compass for every REST endpoint you design.

