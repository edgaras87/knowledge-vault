---
title: '@Message'
date: 2025-11-10
tags: 
    - spring
    - spring-boot
    - i18n
    - internationalization
    - architecture
    - cheatsheet 
summary: Field injection for localized strings in Spring applications.
aliases:
    - Spring i18n Layer - @Message Cheatsheet
---

# `@Message` — field injection for localized strings

Here’s a pragmatic, copy-paste-ready way to get an **`@Message` qualifier** that injects *localized strings on demand*.

It’s intentionally simple and robust:

* You mark a field with `@Message("bundle.key")`.
* Spring injects a tiny **`LocalizedMessage`** object that can render the string using the current request locale (`LocaleContextHolder`).
* Works great in controllers, advices, and services.
* Keeps your **`Messages`** utility around for general use; this is just a convenience for the common “one key, many calls” case.


## 1) The API (annotation + type)

```java
// src/main/java/com/example/app/i18n/Message.java
package com.example.app.i18n;

import org.springframework.beans.factory.annotation.Qualifier;

import java.lang.annotation.*;

@Qualifier
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Message {
  String value(); // i18n code, e.g. "problem.duplicate-category.title"
}
```

```java
// src/main/java/com/example/app/i18n/LocalizedMessage.java
package com.example.app.i18n;

/** A message you can render in the current locale, with args. */
@FunctionalInterface
public interface LocalizedMessage {
  String render(Object... args);

  /** Optional helper: static factory if you ever need a manual instance. */
  static LocalizedMessage of(java.util.function.Function<Object[], String> f) { return f::apply; }
}
```

> You’ll inject `LocalizedMessage` fields annotated with `@Message("key")`.

---

## 2) The injector (BeanPostProcessor)

```java
// src/main/java/com/example/app/config/MessageInjectionPostProcessor.java
package com.example.app.config;

import com.example.app.i18n.LocalizedMessage;
import com.example.app.i18n.Message;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.i18n.LocaleContextHolder;

import java.lang.reflect.Field;
import java.util.Objects;

/**
 * Injects LocalizedMessage fields annotated with @Message.
 * Keeps it simple: field injection only (constructor users can still inject Messages directly).
 */
@Configuration
public class MessageInjectionPostProcessor implements BeanPostProcessor {

  private final MessageSource ms;

  public MessageInjectionPostProcessor(MessageSource messageSource) {
    this.ms = Objects.requireNonNull(messageSource);
  }

  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    Class<?> type = bean.getClass();
    while (type != null && type != Object.class) {
      for (Field f : type.getDeclaredFields()) {
        var ann = f.getAnnotation(Message.class);
        if (ann == null) continue;
        if (!LocalizedMessage.class.isAssignableFrom(f.getType())) {
          throw new IllegalStateException("@Message can only be used on fields of type LocalizedMessage: " +
              type.getName() + "#" + f.getName());
        }
        var code = ann.value();
        var lm = (LocalizedMessage) args -> {
          var locale = LocaleContextHolder.getLocale();
          // Use code-as-default to avoid exceptions when missing
          return ms.getMessage(code, args, code, locale);
        };
        try {
          f.setAccessible(true);
          f.set(bean, lm);
        } catch (IllegalAccessException e) {
          throw new IllegalStateException("Failed to inject @Message on " + type.getName() + "#" + f.getName(), e);
        }
      }
      type = type.getSuperclass();
    }
    return bean;
  }
}
```

**Why this shape?**

* Zero magic factories or per-key beans.
* No constructor gymnastics.
* Works anywhere Spring manages the bean.
* Locale is taken from `LocaleContextHolder` (set by your `WebConfig`’s `AcceptHeaderLocaleContextResolver`).

---

## 3) Usage examples

### In a controller advice (ProblemDetail)

```java
// src/main/java/com/example/app/web/support/errors/SomeAdvice.java
package com.example.app.web.support.errors;

import com.example.app.i18n.LocalizedMessage;
import com.example.app.i18n.Message;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
class SomeAdvice {

  @Message("problem.duplicate-category.title")
  private LocalizedMessage duplicateTitle;

  @Message("problem.duplicate-category.detail")
  private LocalizedMessage duplicateDetail;

  @ExceptionHandler(DuplicateCategoryException.class)
  ProblemDetail onDup(DuplicateCategoryException ex) {
    var pd = ProblemDetail.forStatus(409);
    pd.setTitle(duplicateTitle.render());
    pd.setDetail(duplicateDetail.render(ex.getName()));
    return pd;
  }
}
```

### In a regular service

```java
@Service
class Greeter {
  @Message("app.greeting")
  private LocalizedMessage greeting;

  String greet(String name) {
    return greeting.render(name); // uses request locale if present; default otherwise
  }
}
```

---

## 4) Properties (bundle) reminder

`src/main/resources/i18n/messages.properties`

```properties
app.greeting=Hello, {0}!
```

`src/main/resources/i18n/problem-messages.properties`

```properties
problem.duplicate-category.title=Category already exists
problem.duplicate-category.detail=A category with name "{0}" already exists.
```

Your existing `MessageSourceConfig` already loads these.

---

## 5) Notes & options

* **Field injection vs constructor:** this processor handles **fields**. For constructor-only classes, inject your existing `Messages` utility and call `msg(code, args)`; it’s one line and plays nicely with tests. If you *really* want constructor parameter support, we can extend this to an `InstantiationAwareBeanPostProcessor` to create arguments for `@Message`-annotated params of type `LocalizedMessage`.
* **Errors on wrong type:** we fail fast if `@Message` appears on a non-`LocalizedMessage` field.
* **Missing keys:** returns the **code** itself (your `MessageSource` is configured with `useCodeAsDefaultMessage=true`). Flip behavior in the lambda if you prefer exceptions.

---

## 6) Tiny test

```java
// src/test/java/com/example/app/config/MessageInjectionPostProcessorTest.java
package com.example.app.config;

import com.example.app.i18n.LocalizedMessage;
import com.example.app.i18n.Message;
import org.junit.jupiter.api.Test;
import org.springframework.context.annotation.*;
import org.springframework.context.support.StaticMessageSource;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

class MessageInjectionPostProcessorTest {

  @Configuration
  static class Cfg {
    @Bean MessageInjectionPostProcessor mipp(org.springframework.context.MessageSource ms) {
      return new MessageInjectionPostProcessor(ms);
    }
    @Bean org.springframework.context.MessageSource ms() {
      var sms = new StaticMessageSource();
      sms.addMessage("app.greeting", Locale.ENGLISH, "Hello, {0}!");
      return sms;
    }
    @Bean Greeter greeter() { return new Greeter(); }
  }

  static class Greeter {
    @Message("app.greeting")
    LocalizedMessage greet;
  }

  @Test
  void injectsLocalizedMessage() {
    var ctx = new AnnotationConfigApplicationContext(Cfg.class);
    var g = ctx.getBean(Greeter.class);
    assertThat(g.greet.render("World")).isEqualTo("Hello, World!");
  }
}
```

---

## 7) Placement recap

```
src/main/java/com/example/app/
├─ config/
│  └─ MessageInjectionPostProcessor.java  # the injector
└─ i18n/
   ├─ Message.java                        # @Message
   └─ LocalizedMessage.java               # render(args...)
```

---

## 8) When to use what

* **`@Message` + `LocalizedMessage`**: you reuse the same key multiple times in a class and want tidy code like `title.render()`.
* **`Messages` utility**: ad-hoc lookups, dynamic keys, constructor-only classes.
* **`MessageCodes` / `ProblemKey`**: avoid stringly-typed keys and align slugs/titles with `/problems/*`.


