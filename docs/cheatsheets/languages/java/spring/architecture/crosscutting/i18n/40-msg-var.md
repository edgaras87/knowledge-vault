---
title: '@MsgVar'
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - i18n
    - internationalization
    - architecture
    - cheatsheet 
summary: Controller method parameter injection for localized strings in Spring applications.
aliases:
    - Spring i18n Layer - @MsgVar Cheatsheet
---

# `@MsgVar` — resolve an i18n key straight into a `String` param

Here’s a clean, **controller-method parameter** injector so you can write:

```java
@GetMapping("/hello/{name}")
String hello(@MsgVar("app.greeting") String greeting,
             @PathVariable String name) {
  return greeting.formatted(name); // "Hello, Edgaras!"
}
```

No boilerplate, locale-aware per request, and zero surprises.

---



## What it does

* Works on **controller method parameters** of type `String`
* Resolves the message **at request time** (respects `Accept-Language`)
* Optional `args = { ... }` pulls values by name from **path variables** first, then **query params**
* Pairs perfectly with your existing `MessageSourceConfig` & `WebConfig`

---

## 1) The annotation

```java
// src/main/java/com/example/app/i18n/MsgVar.java
package com.example.app.i18n;

import java.lang.annotation.*;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MsgVar {
  /** i18n code, e.g., "app.greeting" or "problem.duplicate-category.title" */
  String value();

  /**
   * Optional argument names to fill {0}, {1}, …
   * Resolution order per name: URI path variables → query parameters.
   * Example: args = {"name","count"}
   */
  String[] args() default {};
}
```

---

## 2) The MVC argument resolver

```java
// src/main/java/com/example/app/web/support/i18n/MsgVarArgumentResolver.java
package com.example.app.web.support.i18n;

import com.example.app.i18n.MsgVar;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.core.MethodParameter;
import org.springframework.http.server.ServletServerHttpRequest;
import org.springframework.lang.Nullable;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.servlet.HandlerMapping;

import jakarta.servlet.http.HttpServletRequest;
import java.util.Locale;
import java.util.Map;
import java.util.Objects;

public class MsgVarArgumentResolver implements HandlerMethodArgumentResolver {

  private final MessageSource ms;

  public MsgVarArgumentResolver(MessageSource ms) {
    this.ms = Objects.requireNonNull(ms);
  }

  @Override
  public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(MsgVar.class)
        && String.class.isAssignableFrom(parameter.getParameterType());
  }

  @Override
  public Object resolveArgument(MethodParameter parameter,
                                @Nullable ModelAndViewContainer mavContainer,
                                NativeWebRequest webRequest,
                                @Nullable WebDataBinderFactory binderFactory) {
    var ann = parameter.getParameterAnnotation(MsgVar.class);
    assert ann != null;

    HttpServletRequest req = webRequest.getNativeRequest(HttpServletRequest.class);
    Map<String, String> pathVars = pathVars(req);
    String[] argNames = ann.args();
    Object[] args = new Object[argNames.length];
    for (int i = 0; i < argNames.length; i++) {
      String name = argNames[i];
      String v = (pathVars != null) ? pathVars.get(name) : null;
      if (v == null) v = req.getParameter(name); // fallback to query param
      args[i] = v; // may be null; MessageSource will stringize
    }

    Locale locale = LocaleContextHolder.getLocale();
    // Use code-as-default to avoid exceptions; flip to `null` for strict mode
    return ms.getMessage(ann.value(), args, ann.value(), locale);
  }

  @SuppressWarnings("unchecked")
  private Map<String, String> pathVars(HttpServletRequest req) {
    return (Map<String, String>) req.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);
  }
}
```

---

## 3) Wire it into MVC

```java
// src/main/java/com/example/app/config/I18nWebConfig.java
package com.example.app.config;

import com.example.app.web.support.i18n.MsgVarArgumentResolver;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.List;

@Configuration
public class I18nWebConfig implements WebMvcConfigurer {
  private final MessageSource ms;
  public I18nWebConfig(MessageSource ms) { this.ms = ms; }

  @Override
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(new MsgVarArgumentResolver(ms));
  }
}
```

> This sits alongside your existing `WebConfig`. Keeping it separate avoids mixing concerns.

---

## 4) Usage examples

### a) Path variable → `{0}`

```java
@GetMapping("/hello/{name}")
public String hello(@MsgVar("app.greeting") String greeting,
                    @PathVariable String name) {
  return greeting.formatted(name); // "Hello, Edgaras!"
}
```

Where `i18n/messages.properties` contains:

```properties
app.greeting=Hello, {0}!
```

### b) Named args from path/query (order matters)

```java
@GetMapping("/report/{year}")
public Map<String,Object> report(
    @MsgVar(value="app.report.title", args={"year","region"}) String title,
    @PathVariable int year,
    @RequestParam(defaultValue="eu") String region) {
  return Map.of("title", title, "year", year, "region", region);
}
```

`i18n/messages.properties`

```properties
app.report.title=Report {0} ({1})
```

### c) Problem docs header (no args)

```java
@GetMapping(value="/problems", produces="text/html")
public String problems(@MsgVar("app.problems.index.title") String pageTitle) {
  return "<h1>"+pageTitle+"</h1>";
}
```

---

## 5) Behavior & guardrails

* Resolution happens **per request**, so locale is always correct.
* `args` are pulled by **name**: first from path variables, then from query params.
* Missing keys return the **code** string (dev-friendly). For strict prod, change the resolver’s `getMessage` default from `ann.value()` to `null` to throw.

---

## 6) Optional: strict mode via property

If you want an on/off switch:

```java
// tweak in resolver:
boolean strict = Boolean.parseBoolean(
    webRequest.getHeader("X-I18N-Strict") // or read from env/config
);
String def = strict ? null : ann.value();
return ms.getMessage(ann.value(), args, def, locale);
```

---

## 7) Mini tests

```java
// src/test/java/com/example/app/web/support/i18n/MsgVarArgumentResolverTest.java
package com.example.app.web.support.i18n;

import com.example.app.config.I18nWebConfig;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.support.StaticMessageSource;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.util.Locale;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(controllers = MsgVarArgumentResolverTest.Ctrl.class)
@Import(I18nWebConfig.class)
class MsgVarArgumentResolverTest {

  @Autowired MockMvc mvc;

  @org.springframework.context.annotation.Configuration
  static class Cfg {
    @Bean StaticMessageSource messageSource() {
      var sms = new StaticMessageSource();
      sms.addMessage("app.greeting", Locale.ENGLISH, "Hello, {0}!");
      return sms;
    }
  }

  @RestController
  static class Ctrl {
    @GetMapping("/hello/{name}")
    String hello(@com.example.app.i18n.MsgVar("app.greeting") String greeting,
                 @PathVariable String name) {
      return greeting.formatted(name);
    }
  }

  @Test
  void injectsLocalizedString() throws Exception {
    mvc.perform(get("/hello/World").header("Accept-Language", "en"))
       .andExpect(status().isOk())
       .andExpect(content().string("Hello, World!"));
  }
}
```

---

## 8) Placement summary

```
src/main/java/com/example/app/
├─ i18n/
│  └─ MsgVar.java
├─ web/support/i18n/
│  └─ MsgVarArgumentResolver.java
└─ config/
   └─ I18nWebConfig.java            # adds resolver to MVC
```

---

### What about constructor/setter params?

You already have them via the **MessageAutowireResolver** we built earlier:

* `@Msg("key") LocalizedMessage` on **constructor or setter params** → injected supplier.
* `@MsgVar("key") String` is **only for controller method parameters** (request-scoped resolution).

If you want `@MsgVar` to also work on **handler method return values** (e.g., auto-wrap a `String` return), we can add a `HandlerMethodReturnValueHandler` next.
