---
title: Pagination
date: 2025-10-22
tags: 
    - java
    - spring
    - data
    - pagination
summary: A cheatsheet for Spring Data's pagination features, covering Page, Pageable, PageRequest, and best practices for implementing offset pagination in Spring applications.
aliases:
  - Spring Data Pagination
---

# Spring Data Pagination — `Page`, `Pageable`, `PageRequest` (Cheatsheet)

> Offset pagination done right. Clean endpoints, predictable metadata, no hand-rolled foot-guns.

---

## TL;DR

* **`Page<T>`** = results **+** metadata (total, pages, etc.).
* **`Pageable`** = request (page, size, sort).
* **`PageRequest.of(page, size, sort...)`** builds a `Pageable`.
* **`Slice<T>`** = cheaper cousin when you don’t need totals.

---

## Core Types (where they live)

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
```

---

## Building a `PageRequest`

```java
int page = Math.max(0, requestedPage);     // zero-based
int size = Math.min(Math.max(requestedSize, 1), 100); // clamp (e.g., 1..100)

Sort sort = Sort.by(
  Sort.Order.desc("createdAt"),
  Sort.Order.asc("lastName")
);

PageRequest pr = PageRequest.of(page, size, sort);
```

Common one-liners:

```java
PageRequest.of(0, 20);
PageRequest.of(page, size, Sort.by("lastName").ascending());
PageRequest.of(page, size, Sort.by(Sort.Order.desc("createdAt")));
```

---

## Repository patterns

```java
public interface ContactRepository extends JpaRepository<Contact, Long> {
  Page<Contact> findAll(Pageable pageable); // built-in
  Page<Contact> findByLastNameContainingIgnoreCase(String q, Pageable pageable);
}
```

---

## Controller patterns

### 1) Return `Page<T>` directly (quickest)

```java
@GetMapping("/contacts")
public Page<Contact> list(@PageableDefault(size = 20, sort = "id") Pageable pageable) {
  return repo.findAll(pageable);
}
// supports: ?page=0&size=50&sort=lastName,desc&sort=createdAt,asc
```

### 2) Wrap into DTO + meta (client-friendly)

```java
record PageMeta(int page, int size, long totalElements, int totalPages, boolean first, boolean last) {}
record PageResponse<T>(List<T> items, PageMeta meta) {}

@GetMapping("/contacts")
public ResponseEntity<PageResponse<ContactDto>> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size) {

  PageRequest pr = PageRequest.of(
      Math.max(0, page),
      Math.min(Math.max(size, 1), 100),
      Sort.by(Sort.Order.desc("createdAt"))
  );

  Page<ContactDto> p = repo.findAll(pr).map(ContactDto::from);

  var body = new PageResponse<>(
      p.getContent(),
      new PageMeta(p.getNumber(), p.getSize(), p.getTotalElements(), p.getTotalPages(), p.isFirst(), p.isLast())
  );

  return ResponseEntity.ok(body);
}
```

### 3) Add pagination headers (for tables/infinite scroll UIs)

```java
@GetMapping("/contacts")
public ResponseEntity<List<ContactDto>> list(Pageable pageable) {
  Page<ContactDto> p = repo.findAll(pageable).map(ContactDto::from);

  return ResponseEntity.ok()
      .header("X-Total-Count", String.valueOf(p.getTotalElements()))
      .header("X-Total-Pages", String.valueOf(p.getTotalPages()))
      .header("X-Page", String.valueOf(p.getNumber()))
      .header("X-Size", String.valueOf(p.getSize()))
      .body(p.getContent());
}
```

Optional: RFC-5988 `Link` header for navigation:

```
Link: </contacts?page=0&size=20>; rel="first",
      </contacts?page=3&size=20>; rel="prev",
      </contacts?page=5&size=20>; rel="next",
      </contacts?page=12&size=20>; rel="last"
```

---

## Sorting: safe, explicit

```java
Sort sort = Sort.by(
  Sort.Order.desc("createdAt").ignoreCase(), // ignoreCase only applies to strings
  Sort.Order.asc("lastName")
);
PageRequest pr = PageRequest.of(page, size, sort);
```

Multiple `sort` params are supported out of the box:

```
/contacts?sort=lastName,desc&sort=createdAt,asc
```

---

## Validating `page` / `size`

* `page` is **zero-based**.
* Clamp `size` to a sane range (e.g., 1..100) to avoid DOS-by-oversized pages.
* Consider a global clamp:

```java
@RestControllerAdvice
class PagingAdvice {
  @InitBinder
  void clamp(WebDataBinder binder) {
    binder.registerCustomEditor(Integer.class, "size", new PropertyEditorSupport() {
      @Override public void setAsText(String text) {
        int v = Integer.parseInt(text);
        setValue(Math.min(Math.max(v, 1), 100));
      }
    });
  }
}
```

---

## Mapping Entities → DTOs (without N+1 drama)

```java
Page<ContactDto> page = repo.findAll(pr).map(ContactDto::from);
List<ContactDto> items = page.getContent();
```

If DTO needs joined fields, prefer JPA projections or fetch joins in the repository to avoid lazy loading per row.

---

## `Slice<T>` for infinite scroll

* No `COUNT(*)` → faster on large datasets.
* Still has `hasNext()`.

```java
Slice<Contact> slice = repo.findAllByActiveTrue(PageRequest.of(page, size));
boolean more = slice.hasNext();
```

---

## Performance notes (don’t learn these the hard way)

* **COUNT(*) can dominate.** On huge tables, `Page<T>` can be expensive. If you don’t need totals, switch to `Slice<T>`.
* **Index your sort keys.** Sorting on unindexed columns = slow.
* **Deterministic sorting.** Add a tiebreaker (e.g., `createdAt desc, id desc`) for stable paging.
* **Offset pagination drifts** when rows are inserted/deleted between pages. For ultra-stable feeds, consider **keyset (seek) pagination** later.

---

## Error handling & edge cases

* Out-of-range page numbers still return `content: []` with valid meta. That’s fine.
* Do **not** return 204 for an empty page; clients expect paging metadata.
* Validate sort properties against a whitelist if you expose user-driven sorts.

---

## Tests (WebMvc)

```java
@WebMvcTest(ContactResource.class)
class ContactResourcePagingTests {
  @Autowired MockMvc mvc;
  @MockBean ContactRepository repo;

  @Test
  void list_paginated_ok() throws Exception {
    PageRequest pr = PageRequest.of(0, 2, Sort.by("id").ascending());
    Page<Contact> page = new PageImpl<>(List.of(new Contact(1L,"A"), new Contact(2L,"B")), pr, 5);

    when(repo.findAll(any(Pageable.class))).thenReturn(page);

    mvc.perform(get("/contacts?page=0&size=2&sort=id,asc"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.items.length()").value(2))
       .andExpect(jsonPath("$.meta.totalElements").value(5));
  }
}
```

---

## Anti-patterns to avoid

* Building `PageRequest` from raw strings without clamping.
* Returning `List<T>` with separate “count” endpoint — brittle and chatty.
* Accepting a comma-delimited `sort` string and manually parsing it when `Pageable` already handles multi-sort.
* Doing DTO mapping *after* the transaction closes if you rely on lazy associations.

---

## Copy-paste templates

**Simple endpoint with `Pageable` injection**

```java
@GetMapping("/contacts")
public Page<ContactDto> list(@PageableDefault(size = 20, sort = "id") Pageable pageable) {
  return repo.findAll(pageable).map(ContactDto::from);
}
```

**Explicit clamped params + DTO wrapper**

```java
@GetMapping("/contacts")
public ResponseEntity<PageResponse<ContactDto>> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size) {

  page = Math.max(0, page);
  size = Math.min(Math.max(size, 1), 100);

  Page<ContactDto> p = repo.findAll(PageRequest.of(page, size, Sort.by("id").ascending()))
                           .map(ContactDto::from);

  return ResponseEntity.ok(new PageResponse<>(
      p.getContent(),
      new PageMeta(p.getNumber(), p.getSize(), p.getTotalElements(), p.getTotalPages(), p.isFirst(), p.isLast())
  ));
}
```

---

## When to graduate beyond `Page<T>`

* **Timelines / streams** where order is append-only → **keyset (seek) pagination**.
* **Very large datasets** where global counts are expensive → **Slice<T>** or **approximate counts**.
* **API contracts** that need stable cursors → **cursor pagination**.

---

### Naming recap

* Use: `cheatsheets/frameworks/spring/data/pageable.md`
* If you plan sections like `slice.md`, `keyset-pagination.md`, keep this file focused on offset paging; otherwise rename to `pagination.md` and add sections over time.

