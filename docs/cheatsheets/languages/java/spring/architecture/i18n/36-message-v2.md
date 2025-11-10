---
title: '@Message v2' 
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
    - Spring i18n Layer - @Message v2 Cheatsheet
---

# Message injection v2 — fields, setters, and constructors (+ `@Msg` alias)

---

Here **`@Message`** to work on **constructor params** and **setter params**, plus a short alias **`@Msg`**. Here’s a clean, production-ready bundle that does all three while keeping your existing field injector intact.

## What you get

* `@Message("i18n.key")` (or `@Msg("i18n.key")`) on:

  * **fields** (as before)
  * **setter parameters**
  * **constructor parameters**
* Injected value is a `LocalizedMessage` you call like `title.render(arg0, arg1, ...)`.
* Respects request locale via `LocaleContextHolder`.

---

## 1) Annotations (`@Message` + `@Msg` alias)

```java
// src/main/java/com/example/app/i18n/Message.java
package com.example.app.i18n;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.annotation.AliasFor;

import java.lang.annotation.*;

@Qualifier
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Message {
  @AliasFor("code")
  String value() default "";
  @AliasFor("value")
  String code() default "";
}
```

```java
// src/main/java/com/example/app/i18n/Msg.java
package com.example.app.i18n;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.annotation.AliasFor;

import java.lang.annotation.*;

@Qualifier
@Message // <- meta-annotated, so it behaves like @Message
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Msg {
  @AliasFor(annotation = Message.class, attribute = "value")
  String value();
}
```

```java
// src/main/java/com/example/app/i18n/LocalizedMessage.java
package com.example.app.i18n;

@FunctionalInterface
public interface LocalizedMessage {
  String render(Object... args);
}
```

---

## 2) Field injector (kept, works without @Autowired)

Your existing **field** convenience stays, unchanged:

```java
// src/main/java/com/example/app/config/MessageInjectionPostProcessor.java
package com.example.app.config;

import com.example.app.i18n.LocalizedMessage;
import com.example.app.i18n.Message;
import com.example.app.i18n.Msg;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.core.annotation.AnnotationUtils;

import java.lang.reflect.Field;
import java.util.Objects;

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
        var ann = AnnotationUtils.getAnnotation(f, Message.class);
        if (ann == null) ann = AnnotationUtils.getAnnotation(f, Msg.class);
        if (ann == null) continue;

        if (!LocalizedMessage.class.isAssignableFrom(f.getType())) {
          throw new IllegalStateException("@Message/@Msg allowed only on LocalizedMessage fields: " +
              type.getName() + "#" + f.getName());
        }

        final String code = (ann instanceof Message m) ? m.code()
                           : ((Msg) ann).value();
        LocalizedMessage lm = args -> ms.getMessage(code, args, code, LocaleContextHolder.getLocale());

        try { f.setAccessible(true); f.set(bean, lm); }
        catch (IllegalAccessException e) {
          throw new IllegalStateException("Failed to inject @Message on " + type.getName() + "#" + f.getName(), e);
        }
      }
      type = type.getSuperclass();
    }
    return bean;
  }
}
```

---

## 3) **Constructor + setter parameter** injection

We hook into Spring’s autowiring by adding a **custom autowire candidate resolver**.
It creates a `LocalizedMessage` when the dependency **type is `LocalizedMessage`** and the parameter is annotated with `@Message` or `@Msg`.

```java
// src/main/java/com/example/app/config/MessageAutowireResolverConfig.java
package com.example.app.config;

import com.example.app.i18n.LocalizedMessage;
import com.example.app.i18n.Message;
import com.example.app.i18n.Msg;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.SmartInitializingSingleton;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.beans.factory.config.DependencyDescriptor;
import org.springframework.beans.factory.support.AutowireCandidateResolver;
import org.springframework.beans.factory.annotation.QualifierAnnotationAutowireCandidateResolver;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.core.MethodParameter;
import org.springframework.core.annotation.AnnotationUtils;

import java.lang.annotation.Annotation;

@Configuration
public class MessageAutowireResolverConfig implements BeanFactoryPostProcessor, SmartInitializingSingleton {

  private DefaultListableBeanFactory bf;

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    this.bf = (DefaultListableBeanFactory) beanFactory;
    AutowireCandidateResolver current = bf.getAutowireCandidateResolver();
    var enhanced = new MessageAwareResolver(current, bf);
    bf.setAutowireCandidateResolver(enhanced);
  }

  @Override
  public void afterSingletonsInstantiated() { /* no-op */ }

  static final class MessageAwareResolver extends QualifierAnnotationAutowireCandidateResolver {
    private final AutowireCandidateResolver delegate;
    private final DefaultListableBeanFactory bf;

    MessageAwareResolver(AutowireCandidateResolver delegate, DefaultListableBeanFactory bf) {
      this.delegate = delegate;
      this.bf = bf;
      // Let @Qualifier keep working:
      setBeanFactory(bf);
    }

    @Override
    public Object resolveDependency(DependencyDescriptor descriptor, String beanName,
                                    ConfigurableListableBeanFactory beanFactory,
                                    java.util.Set<String> autowiredBeanNames) {
      // Only for LocalizedMessage
      if (LocalizedMessage.class.isAssignableFrom(descriptor.getDependencyType())) {
        // Try to read @Message or @Msg from the annotated element
        String code = extractCode(descriptor);
        if (code != null && !code.isBlank()) {
          // Lazy-fetch MessageSource so early init order isn’t a problem
          MessageSource ms = bf.getBean(MessageSource.class);
          return (LocalizedMessage) args ->
              ms.getMessage(code, args, code, LocaleContextHolder.getLocale());
        }
      }
      // Fallback to default behavior
      return (delegate != null)
          ? delegate.resolveDependency(descriptor, beanName, beanFactory, autowiredBeanNames)
          : super.resolveDependency(descriptor, beanName, beanFactory, autowiredBeanNames);
    }

    private String extractCode(DependencyDescriptor dd) {
      // Constructor / method parameter annotations
      MethodParameter param = dd.getMethodParameter();
      if (param != null) {
        String c = read(param.getParameterAnnotations());
        if (c != null) return c;
      }
      // Field annotations (setter injection ultimately ends up here too)
      Annotation[] anns = dd.getAnnotatedElement() != null
          ? dd.getAnnotatedElement().getAnnotations()
          : null;
      return read(anns);
    }

    private String read(Annotation[] anns) {
      if (anns == null) return null;
      for (Annotation a : anns) {
        Message m = AnnotationUtils.getAnnotation(a, Message.class);
        if (m != null) return m.code();
        Msg msg = AnnotationUtils.getAnnotation(a, Msg.class);
        if (msg != null) return msg.value();
      }
      return null;
    }
  }
}
```

**What this enables**

```java
// Constructor parameter injection
@RestController
class HelloController {
  private final LocalizedMessage greeting;

  public HelloController(@Msg("app.greeting") LocalizedMessage greeting) {
    this.greeting = greeting;
  }

  @GetMapping("/hello/{name}")
  String hello(@PathVariable String name) {
    return greeting.render(name);
  }
}
```

```java
// Setter parameter injection
@Service
class Greeter {
  private LocalizedMessage greeting;

  @Autowired
  void setGreeting(@Message("app.greeting") LocalizedMessage greeting) {
    this.greeting = greeting;
  }
}
```

---

## 4) Quick test sketch

```java
// src/test/java/com/example/app/config/MessageAutowireResolverConfigTest.java
package com.example.app.config;

import com.example.app.i18n.LocalizedMessage;
import com.example.app.i18n.Msg;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.*;
import org.springframework.context.support.StaticMessageSource;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

@SpringJUnitConfig(classes = MessageAutowireResolverConfigTest.Cfg.class)
class MessageAutowireResolverConfigTest {

  @Configuration
  @Import({MessageInjectionPostProcessor.class, MessageAutowireResolverConfig.class})
  static class Cfg {
    @Bean StaticMessageSource messageSource() {
      var sms = new StaticMessageSource();
      sms.addMessage("app.greeting", Locale.ENGLISH, "Hello, {0}!");
      return sms;
    }
    @Bean TestBean testBean(@Msg("app.greeting") LocalizedMessage m) { return new TestBean(m); }
  }

  static class TestBean {
    final LocalizedMessage greet;
    @Autowired
    TestBean(@Msg("app.greeting") LocalizedMessage greet) { this.greet = greet; }
  }

  @Autowired TestBean bean;

  @Test
  void injectsIntoConstructorParam() {
    assertThat(bean.greet.render("World")).isEqualTo("Hello, World!");
  }
}
```

---

## 5) Usage tips

* **Fields:** no `@Autowired` needed; the field injector handles it.
* **Constructors/Setters:** annotate the parameter with `@Message` or `@Msg`; Spring’s autowire resolver will supply a `LocalizedMessage`.
* **Missing keys:** you’ll get the **code** back (dev-friendly). If you want fail-fast in prod, change both lambdas to use `getMessage(code, args, /* defaultMessage */ null, locale)` and let Spring throw on missing keys.

---

## 6) Placement summary

```
src/main/java/com/example/app/
├─ i18n/
│  ├─ LocalizedMessage.java
│  ├─ Message.java
│  └─ Msg.java
└─ config/
   ├─ MessageInjectionPostProcessor.java     # fields
   └─ MessageAutowireResolverConfig.java     # constructor + setter params
```

This keeps **API** in `i18n/` and **wiring** in `config/`, matching the rest of your architecture.

Also add a **`@MsgVar("key")`** parameter annotation for *method arguments* (e.g., inject a resolved `String` immediately rather than a `LocalizedMessage`)?
