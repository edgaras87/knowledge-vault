---
title: Locale Switching 
date: 2025-11-07
tags: 
    - java
    - spring
    - i18n
    - internationalization
    - cheatsheet
summary: A quick reference on locale switching in Spring applications, covering how Spring determines the request locale and how to customize locale resolution for internationalization (i18n).
aliases:
    - Spring i18n - Locale Switching Cheatsheet
---

# Locale Switching — How Spring Chooses Language

Message bundles only matter once you answer one question:

- **Which language should the application speak for this request?**

Spring provides a clean mechanism for that, built around a simple idea:

- **Every request has a `Locale`.**

Everything i18n downstream depends on this value:

- Bean Validation messages  
- `messageSource.getMessage(...)`  
- ProblemDetail `title`/`detail`  
- static i18n docs  
- custom error messages

This file explains how Spring decides that locale and how you can influence it.

---

## The default: `Accept-Language` header

If you do nothing, Spring uses:

```
Accept-Language: de-DE,de;q=0.9,en-US;q=0.8
```

Spring looks at the highest-ranked language and selects the nearest matching bundle.

Client asks German → German bundle used.  
Client asks Polish → fallback to default bundle.  
Client sends nothing → default locale.

This works out-of-the-box thanks to `AcceptHeaderLocaleResolver`.

For most APIs, this is *already enough*.

---

## Explicit configuration (when you need more control)

Sometimes you want:

- fixed default locale  
- to ignore system locale  
- to enforce one locale per environment  
- to allow clients to switch language explicitly  

Then you configure your own resolver:

```java
@Configuration
public class LocaleConfig {

    @Bean
    public LocaleResolver localeResolver() {
        var resolver = new AcceptHeaderLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }
}
```

This gives you full control without changing client behavior.

---

## Using cookies or sessions (UI-backed apps)

If you ever add a UI or need user-specific settings, you can switch to:

```java
@Bean
public LocaleResolver localeResolver() {
    return new CookieLocaleResolver();  // or SessionLocaleResolver
}
```

Then your app can set locale via endpoints like:

```
POST /locale?lang=de
```

This is *not* common in pure REST APIs but good to know for future growth.

---

## Custom locale extraction (power feature)

When your app needs something non-standard — like language encoded in:

* JWT claims
* user preferences stored in DB
* URL path (`/fr/products/...`)
* tenant metadata

—you implement `LocaleResolver` yourself:

```java
public class JwtLocaleResolver implements LocaleResolver {

    @Override
    public Locale resolveLocale(HttpServletRequest req) {
        String lang = extractFromJwt(req);
        return Locale.forLanguageTag(lang != null ? lang : "en");
    }

    @Override
    public void setLocale(HttpServletRequest req, HttpServletResponse res, Locale loc) {
        // usually no-op in stateless APIs
    }
}
```

This is how multi-region enterprise systems do it.

---

## How locale flows through the request

Once the resolver chooses a locale:

1. Spring stores it in `LocaleContextHolder`.
2. Any code inside the request can access it.
3. Message lookups automatically use it.

This provides a *clean pipeline* from HTTP → messages → response.

Example:

```java
String text = messages.getMessage("user.email.invalid", null, LocaleContextHolder.getLocale());
```

You don’t pass locale around manually — it’s ambient.

---

## Interaction with Bean Validation

Validation messages resolve automatically with locale:

```java
@NotBlank(message = "{user.email.required}")
```

When validation fails:

* Spring collects the violations
* It resolves each message using the active locale
* Your handler produces localized errors in `ProblemDetail.errors`

All without extra code.

---

## Interaction with ProblemDetail

Your exception handler may produce:

```java
problemDetail.setDetail(
    messages.getMessage("category.duplicate", null, LocaleContextHolder.getLocale())
);
```

When your API grows multilingual, this is the key link:

**locale → messageSource → ProblemDetail → API clients**

A single `LocaleResolver` influences the whole chain.

---

## Locale and static docs (`/static/problems/*.html`)

If you eventually produce localized ProblemDetail HTML pages:

```
src/main/resources/static/problems/
    duplicate-category.en.html
    duplicate-category.de.html
```

You can route to them by locale before rendering.

The resolver still drives the entire experience.

---

## Testing locale switching

Example test:

```java
mockMvc.perform(post("/users")
    .content("{}")
    .contentType(APPLICATION_JSON)
    .header("Accept-Language", "de"))
  .andExpect(jsonPath("$.errors[0].message")
      .value("E-Mail ist erforderlich."));
```

This confirms that your system:

* selected German locale
* loaded German bundle
* resolved German validation message
* delivered localized ProblemDetail

This is exactly the level of confidence you want in production-grade APIs.

---

## When to introduce locale switching

Introduce it when:

* you want to future-proof error messages
* you start adding Slovak/German/English variants
* teams work in multiple regions
* your Product Owner wants proper “human errors”

Don't introduce it before you're ready — but once it's there, it becomes invisible infrastructure that makes everything cleaner.

---

## This completes the i18n triad

You now have:

1. **Message bundles** — where translations live
2. **Validation messages** — how Bean Validation bridges into i18n
3. **Locale switching** — how Spring decides language

This is enough to run a production-ready multilingual backend with consistent, predictable behavior across your entire architecture.

