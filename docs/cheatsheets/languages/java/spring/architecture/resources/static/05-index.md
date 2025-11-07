---
title: Content 
date: 2025-11-07
tags: 
    - java
    - spring
    - architecture
    - static-assets
    - cheatsheet
summary: A quick reference on using Spring Boot's static assets feature to serve public files, API docs, and problem documentation efficiently and effectively.
aliases:
  - Spring Static Assets - Content Cheatsheet
---

# Static Assets — Index

`src/main/resources/static/` is your app’s **dumb file server**: no controllers, no templating — just bytes.  

Spring Boot auto-exposes files here under `/<filename>` with smart defaults, so you can ship docs and assets without writing code.

This page gives you the mental model and the rules; details live in `serving-static.md` and `problem-docs.md`.

---

## What belongs in `static/` (more than HTML)

- **Problem docs** for RFC-7807 (`/problems/*.html`)
- **API docs (OpenAPI)**: `openapi.json` or `openapi.yaml` → `/openapi.json`
- **Docs site exports** (prebuilt HTML)
- **Images, CSS, JS** for any static pages
- **Well-known files**:
  - `robots.txt`
  - `security.txt` (`/.well-known/security.txt`)
  - `apple-app-site-association`, `assetlinks.json`
- **Favicons / manifest**: `/favicon.ico`, `/site.webmanifest`
- **Landing/healthy “it works” page**: `/index.html` at the root
- **Downloadable artifacts** (e.g., PDFs) that don’t need auth

If it’s static and public and doesn’t need request context, it probably belongs here.

---

## Where Spring serves from (and in what order)

By default, Spring checks these classpath locations (first match wins):

```
/static/
/public/
/resources/
/META-INF/resources/
```

The common one you’ll use:  
`src/main/resources/static/ → https://host/<file>`

You can override locations via `spring.web.resources.static-locations` if needed.

---

## URL mapping rules you’ll actually feel

- `static/index.html` becomes `/` (root) if present.
- `static/docs/index.html` becomes `/docs/`.
- Nested folders map directly: `static/problems/duplicate-category.html` → `/problems/duplicate-category.html`.
- If you link without extension, prefer consistent URLs (`/problems/duplicate-category` vs `/...html`) and stick to one style.

Pro tip: for problem docs, use **extensionless URIs** as your canonical “type” and keep the file as `.html`.

---

## Caching & versioning (production realism)

Static assets should be cached hard in prod.

**YAML (`application-prod.yml`) example:**
```yaml
spring:
  web:
    resources:
      cache:
        period: 365d
      chain:
        enabled: true
        strategy:
          content:
            enabled: true
            paths: "/**"   # add content hash to URLs
```

Then reference assets with content-versioned paths (Spring will rewrite links in templating contexts; for raw HTML, build tools should fingerprint filenames).

For docs pages like `/problems/*`, keep **no-cache** if you want instant updates; for CSS/JS/images, **cache forever** + fingerprint.

---

## Security & routing considerations

* Files in `static/` are **public** if you don’t secure them. If you need to hide something, **don’t** put it in `static/`.
* Spring Security usually lets static assets bypass auth (`/css/**`, `/js/**`, `/images/**`, `/problems/**`). Verify your matcher rules.
* If you later add SPA routing, ensure `/index.html` fallback doesn’t hijack API routes.

---

## When **not** to use `static/`

* Anything that requires **user context** or **authorization**.
* Dynamic pages (Thymeleaf/Freemarker) → use templates + controllers.
* Files that must be **private** or **generated per user** (serve them via a controller with proper auth).

---

## Suggested structure (scales cleanly)

```
src/main/resources/static/
├─ index.html
├─ problems/                 # RFC-7807 human docs
│  ├─ duplicate-category.html
│  └─ invalid-input.html
├─ docs/                     # API or product docs (prebuilt)
│  └─ index.html
├─ openapi.json
├─ .well-known/
│  └─ security.txt
├─ assets/
│  ├─ css/
│  ├─ js/
│  └─ img/
├─ favicon.ico
└─ site.webmanifest
```

Keep **docs** and **assets** separate so problem pages can be minimal and stable.

---

## Local dev vs prod

* Devtools will reload changed static files without restarts.
* In dev, prefer low/no cache; in prod, fingerprint + long cache for assets.
* If you serve docs behind a CDN, ensure headers / cache rules are consistent between app and edge.

---

## Cross-links with your other cheatsheets

* **Problem docs**: see `static-assets/problem-docs.md` for RFC-7807 alignment.
* **i18n**: if you localize docs, consider language-specific file variants and a small router controller (optional).
* **Logging**: serve a “how to report issues” page and link it from problem docs.
* **App config**: tweak `spring.web.resources.*` in `application-*.yml` as you go.

---

## TL;DR

Yes, you want an `index.md` here.
`static/` is a first-class delivery mechanism for API docs, problem pages, and public assets.
Use it deliberately: public, cacheable, boring — and rock solid in production.
