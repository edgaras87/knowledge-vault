---
title: Problem Docs
date: 2025-11-07
tags: 
    - java
    - spring
    - architecture
    - static-assets
    - cheatsheet
summary: A quick reference on creating and serving static RFC-7807 problem documentation pages in Spring applications to provide meaningful error context for API clients.
aliases:
  - Spring Static Assets - Problem Docs Cheatsheet
---

# Problem Docs — Static RFC-7807 Pages

Your API should speak in stable, predictable errors. RFC-7807 gives you a structure; your static `/problems/*` pages give those structures *meaning*.

This file explains how to create, organize, and serve static problem documentation that your API clients — and your future self — can rely on.

---

## Why static problem docs exist

A `ProblemDetail` response carries a `type`:

```
"type": "[https://api.example.com/problems/duplicate-category](https://api.example.com/problems/duplicate-category)"
```

Clients need to know what that type means:

- When does this error occur?  
- What fields are relevant?  
- Is it retryable?  
- Can the user fix it?  

Static HTML pages provide this context.

This pattern is used by Stripe, GitHub, AWS, and large enterprise APIs because it creates a stable contract between backend and client independent of code.

---

## Where problem docs live

Spring Boot automatically serves static content from:

```

src/main/resources/static/

```

So you place your docs here:

```
src/main/resources/static/problems/
duplicate-category.html
invalid-input.html
unauthorized.html
quota-limit.html
```

No controller needed.  
Spring’s resource handler does the work.

---

## How these docs tie into `ProblemDetail`

When generating a ProblemDetail, you set:

```java
problemDetail.setType(URI.create(problemLinks.type("duplicate-category")));
```

Where `problemLinks.type("duplicate-category")` resolves to:

```
/problems/duplicate-category
```

The client hits that URI → gets a human-readable page explaining the error.

The API stays machine-readable.
The docs stay human-readable.
This separation keeps the entire error system future-proof.

---

## Example problem doc (HTML template)

Your docs don’t have to be fancy — clarity wins over design.

`src/main/resources/static/problems/duplicate-category.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Duplicate Category</title>
</head>
<body>
  <h1>Duplicate Category</h1>
  <p>This error occurs when a category with the same name already exists.</p>

  <h3>When it happens</h3>
  <ul>
    <li>Client attempts to create a category with an existing name.</li>
    <li>A case-insensitive match is performed by the system.</li>
  </ul>

  <h3>How to fix</h3>
  <ul>
    <li>Choose a unique category name.</li>
    <li>Fetch existing categories before submitting.</li>
  </ul>

  <h3>Technical notes</h3>
  <p>Error code: <strong>category.duplicate</strong></p>
  <p>HTTP status: <strong>409 Conflict</strong></p>
</body>
</html>
```

This mirrors what Stripe, Plaid, and Amazon do — simple, informative, stable.

---

## Versioning problem docs (future-proofing)

If your API evolves, you may want versioned problem types:

```
/problems/v1/duplicate-category
/proble ms/v2/duplicate-category
```

Your folder can grow accordingly:

```
static/problems/v1/
static/problems/v2/
```

You only introduce versioning when the semantics *change*, not when the code changes.

---

## i18n and problem docs

You have two strategies:

### 1) Keep docs in a single language

This is typical for backend-only services.

### 2) Localize docs per locale

Example:

```
static/problems/duplicate-category.en.html
static/problems/duplicate-category.de.html
```

Your controller (if you add one later) can select the correct variant based on locale:

```java
Locale locale = LocaleContextHolder.getLocale();
String lang = locale.getLanguage(); // "en", "de"
return "problems/duplicate-category." + lang;
```

You don’t need this now, but the system won’t stop you later.

---

## When to write a problem doc

You don’t create docs for *every* error.

Only for:

* domain errors
* business-logic failures
* persistent rules your clients must understand
* errors that occur realistically in real-world flows
* errors that require client-side action

You don’t need docs for internal exceptions like:

* NullPointerException
* IllegalStateException
* DataIntegrityViolationException
* validation failures (`errors[]` already describes them)

Problem docs are for meaningful external contract errors — not accidents.

---

## Integrating with your `ProblemLinks` helper

Your earlier helper fits perfectly here:

```java
@Component
public class ProblemLinks {
    private final URI base;

    public ProblemLinks(AppProps props) {
        this.base = props.host().resolve("problems/");
    }

    public URI type(String slug) {
        return base.resolve(slug);
    }
}
```

This creates stable links between:

* the API error type
* the static documentation
* the contract your client follows

This helper belongs in the **web/support** layer because it is a boundary concern.

---

## Testing problem docs

You should verify they are served:

```java
mockMvc.perform(get("/problems/duplicate-category"))
  .andExpect(status().isOk())
  .andExpect(content().contentTypeCompatibleWith("text/html"));
```

This gives confidence your docs ship with the app.

In integration tests, you can also verify:

* the HTML contains expected sections
* problem `type` URIs match
* no broken links exist

Testing static docs is simple and saves future confusion.

---

## Why this pattern is future-proof

A good ProblemDetail system has:

* stable identifiers (`type` URIs)
* human-readable documentation for each error
* separation of machine contract (JSON) from human language (HTML)
* easy versioning
* i18n flexibility
* no “guessing” about error meaning

Most junior devs skip this part.
Most senior devs swear by it.
Your API becomes *self-describing*, which eliminates half of your future confusion.

---

## Good next candidates

Now that problem docs are carved, the natural next pages are:

* **logging/** (logback, layouts, env-levels)
* **db-migrations/** (especially `flyway-basics.md`)
* **app-config/overrides-and-env.md** (since you touched env earlier)

The resources tree is coming together beautifully — each chamber now has a clear role.
