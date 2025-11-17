---
title: Sanitization & Validation
date: 2025-10-31
tags: 
    - java
    - spring
    - validation
    - sanitization
    - best-practices
summary: A cheatsheet for building a robust string sanitization and validation pipeline in Java Spring applications, ensuring data integrity and consistency across the system.
aliases: 
  - Sanitization & Validation Pipeline Cheatsheet
---

# ✅ **String Sanitization & Validation Pipeline Cheatsheet**

## 🎯 **Goal**

Keep bad text out of your system, without drowning in boilerplate.

**Principle:**

- Two gates, one truth.
- ✅ Clean + validate at the **API boundary**
- ✅ Enforce invariants in the **domain**
- ❌ Don’t duplicate rules in random places

---

## 🧠 **Why this matters**

Dirty input leads to:

* Broken uniqueness checks (`"Book "` vs `"Book"`)
* Unicode confusion (`NFD` vs `NFC`)
* “Invisible bugs” (zero-width chars)
* Inconsistent behavior across API, CLI, batch jobs, Kafka consumers

---

## 🧱 **Pipeline Overview**

| Layer             | Purpose                                     | What happens                                   |
| ----------------- | ------------------------------------------- | ---------------------------------------------- |
| API Boundary      | User-friendly validation & standard cleanup | Global normalize → Bean Validation → map to VO |
| Domain / Internal | True invariants for all callers             | VO `.of()` enforces rules                      |
| DB                | Final protection                            | Canonical unique indexes, column limits        |

---

## 🧹 **Sanitization Function**

**One source of truth:**

```java
public final class Text {
  private Text() {}
  public static String normalize(String s) {
    if (s == null) return null;
    String t = s.trim()
      .replace("\u200B","").replace("\u200C","").replace("\u200D","");
    return java.text.Normalizer.normalize(t, java.text.Normalizer.Form.NFC);
  }
}
```

---

## 🌐 **API Boundary Rules**

**Global JSON normalizer:**

* Normalize *all* Strings in `@RequestBody`
* Add `@Raw` annotation to skip passwords/tokens

**DTO Validation:**

```java
record CreateCategoryRequest(
  @NotBlank @Size(max=64) String name,
  String description
) {}
```

**Controller:**

```java
@Valid @RequestBody CreateCategoryRequest dto
```

---

## 🧬 **Domain Invariants (Internal Guard)**

Use **Value Objects (VOs)** *only for meaningful concepts* (email, slug, canonical name).

Example:

```java
public record EmailAddress(String value) {
  public static EmailAddress of(String raw) {
    var c = Text.normalize(raw);
    if (c == null || !c.matches("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"))
      throw new IllegalArgumentException("Invalid email");
    return new EmailAddress(c);
  }
}
```

**Application services accept VOs, not raw Strings.**
Internal code **cannot bypass** rules.

---

## 🧠 **Which fields get VOs?**

| Field type                                            | Approach                                      |
| ----------------------------------------------------- | --------------------------------------------- |
| Free text (`description`)                             | Normalize at API only. Stay `String`.         |
| Concept with meaning (`email`, `slug`, `unique name`) | VO (`EmailAddress`, `Slug`, `NormalizedName`) |
| Password/raw tokens                                   | **Never normalize** → mark `@Raw`             |

---

## 🗃️ **Database**

* Unique index on canonical expression:

```sql
CREATE UNIQUE INDEX uq_category_name_ci ON category (lower(name));
```

* Match DB column length to your DTO/VO rule

---

## 🪜 **Learning / Production Ladder**

| Stage                         | Use when            | What you do                                                     |
| ----------------------------- | ------------------- | --------------------------------------------------------------- |
| Prototype / practice          | building fast       | Normalize in service, DTO validation                            |
| Serious app                   | long-term code      | Global JSON normalizer + DTO validation                         |
| Production / domain integrity | correctness matters | Introduce VOs for meaning-heavy fields, enforce via type system |

---

## ⚠️ **What NOT to do**

- ❌ Put sanitize logic in every constructor
- ❌ Only rely on `@PrePersist` / `@PreUpdate`
- ❌ Create `CategoryEmail`, `UserEmail`, etc. (duplication)
- ✅ Create **one** `EmailAddress` reused everywhere

---

## 🔑 **Mental triggers**

When you touch a field, ask:

| Question                                       | If yes                    |
| ---------------------------------------------- | ------------------------- |
| “Will this value matter across the system?”    | Make VO                   |
| “Is this user-entered free text?”              | Just normalize at API     |
| “Would a bad input break logic or uniqueness?” | Enforce via VO + DB index |

---

## 🧩 **Minimal VO list for modern apps**

* `EmailAddress`
* `Slug` / `SafeSlug`
* `NormalizedName` (optional, generic)
* `UserId` / `AccountId` (if wrapping UUID/ULID)

Everything else?
Stay `String`, normalized on input.

---

## 🧭 **In one sentence**

> Clean at the edge for convenience, enforce in the domain for truth, back it with the database for safety.

---

## 🚀 **Starter Pack**

Here’s a **Spring Boot “starter pack”** you can paste into a fresh project. It gives you:

* One canonical normalizer (`Text.normalize`)
* Global JSON String cleanup for `@RequestBody` (`@JsonComponent`)
* `@Raw` opt-out for sensitive fields (passwords/tokens)
* ProblemDetail error handler for clean 400/409s
* Minimal VO examples (optional)
* MapStruct wiring (optional)
* DB guard (unique index example)

Use the parts you need now; the rest is ready when you harden.

---

### 0) Folder skeleton (suggested)

```
src/main/java/com/example/shared/text/Text.java
src/main/java/com/example/shared/jackson/Raw.java
src/main/java/com/example/shared/jackson/SanitizingStringDeserializer.java

src/main/java/com/example/shared/web/GlobalErrors.java

# Optional VOs (add as needed)
src/main/java/com/example/shared/vo/EmailAddress.java
src/main/java/com/example/shared/vo/Slug.java
src/main/java/com/example/shared/vo/NormalizedName.java

# Optional MapStruct glue
src/main/java/com/example/shared/mapstruct/CommonConverters.java
src/main/java/com/example/shared/mapstruct/GlobalMapperConfig.java

# Optional JPA converters (if you store VOs in entities)
src/main/java/com/example/shared/jpa/EmailAttr.java
src/main/java/com/example/shared/jpa/SlugAttr.java
src/main/java/com/example/shared/jpa/NormalizedNameAttr.java

# Example Flyway migration (DB guard)
src/main/resources/db/migration/V1__category_indexes.sql
```

---

### 1) Canonical normalizer

```java
// src/main/java/com/example/shared/text/Text.java
package com.example.shared.text;

import java.text.Normalizer;

public final class Text {
  private Text() {}

  /** Trim, strip common zero-widths, NFC. Null-safe. Idempotent. */
  public static String normalize(String s) {
    if (s == null) return null;
    String t = s.trim()
        .replace("\u200B","")  // ZERO WIDTH SPACE
        .replace("\u200C","")  // ZERO WIDTH NON-JOINER
        .replace("\u200D",""); // ZERO WIDTH JOINER
    return Normalizer.normalize(t, Normalizer.Form.NFC);
  }
}
```

---

### 2) Global JSON String sanitizer (+ `@Raw` opt-out)

```java
// src/main/java/com/example/shared/jackson/Raw.java
package com.example.shared.jackson;

import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Raw { /* mark fields that must not be normalized */ }
```

```java
// src/main/java/com/example/shared/jackson/SanitizingStringDeserializer.java
package com.example.shared.jackson;

import com.example.shared.text.Text;
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.deser.ContextualDeserializer;
import org.springframework.boot.jackson.JsonComponent;

@JsonComponent
public class SanitizingStringDeserializer extends JsonDeserializer<String>
                                         implements ContextualDeserializer {

  private final boolean enabled;

  public SanitizingStringDeserializer() { this(true); }
  private SanitizingStringDeserializer(boolean enabled) { this.enabled = enabled; }

  @Override
  public String deserialize(JsonParser p, DeserializationContext ctxt)
      throws java.io.IOException {
    String raw = p.getValueAsString();
    return enabled ? Text.normalize(raw) : raw;
  }

  @Override
  public JsonDeserializer<?> createContextual(DeserializationContext ctxt, BeanProperty prop)
      throws JsonMappingException {
    if (prop == null) return this; // root strings (rare)
    Raw rawAnn = prop.getAnnotation(Raw.class);
    if (rawAnn == null) rawAnn = prop.getContextAnnotation(Raw.class);
    return (rawAnn != null) ? new SanitizingStringDeserializer(false) : this;
  }
}
```

**Usage example (DTO):**

```java
// password is untouched
public record RegisterRequest(String username, @com.example.shared.jackson.Raw String password) {}
```

---

### 3) ProblemDetail error handler (clean 400/409s)

```java
// src/main/java/com/example/shared/web/GlobalErrors.java
package com.example.shared.web;

import jakarta.validation.ConstraintViolationException;
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestControllerAdvice
public class GlobalErrors {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ProblemDetail onInvalid(MethodArgumentNotValidException e) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    pd.setProperty("fieldErrors", e.getBindingResult().getFieldErrors().stream()
      .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
      .toList());
    return pd;
  }

  @ExceptionHandler(ConstraintViolationException.class)
  public ProblemDetail onConstraint(ConstraintViolationException e) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Constraint violation");
    pd.setProperty("violations", e.getConstraintViolations().stream()
      .map(v -> Map.of("path", v.getPropertyPath().toString(), "message", v.getMessage()))
      .toList());
    return pd;
  }

  @ExceptionHandler(IllegalArgumentException.class)
  public ProblemDetail onIllegalArg(IllegalArgumentException e) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Invalid input");
    pd.setDetail(e.getMessage());
    return pd;
  }

  @ExceptionHandler(DuplicateKeyException.class)
  public ProblemDetail onDuplicate(DuplicateKeyException e) {
    var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setTitle("Conflict");
    pd.setDetail(e.getMessage());
    return pd;
  }

  // Simple custom exception you can throw in services for 409
  public static class DuplicateKeyException extends RuntimeException {
    public DuplicateKeyException(String message) { super(message); }
  }
}
```

---

### 4) Optional VOs (add only when rules matter)

```java
// src/main/java/com/example/shared/vo/EmailAddress.java
package com.example.shared.vo;

import com.example.shared.text.Text;

public record EmailAddress(String value) {
  public static EmailAddress of(String raw) {
    var c = Text.normalize(raw);
    if (c == null || !c.matches("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"))
      throw new IllegalArgumentException("Invalid email");
    return new EmailAddress(c);
  }
}
```

```java
// src/main/java/com/example/shared/vo/Slug.java
package com.example.shared.vo;

import com.example.shared.text.Text;

public record Slug(String value) {
  public static Slug of(String raw) {
    var c = Text.normalize(raw).toLowerCase()
        .replaceAll("[^a-z0-9-]+","-").replaceAll("^-+|-+$","");
    if (c.isBlank() || c.length() > 80)
      throw new IllegalArgumentException("Invalid slug");
    return new Slug(c);
  }
}
```

```java
// src/main/java/com/example/shared/vo/NormalizedName.java
package com.example.shared.vo;

import com.example.shared.text.Text;

public record NormalizedName(String value) {
  public static NormalizedName of(String raw) {
    var c = Text.normalize(raw);
    if (c == null || c.isBlank())
      throw new IllegalArgumentException("Name must not be blank");
    return new NormalizedName(c);
  }
}
```

---

### 5) Optional MapStruct wiring (type-based reuse)

```java
// src/main/java/com/example/shared/mapstruct/CommonConverters.java
package com.example.shared.mapstruct;

import com.example.shared.vo.*;
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface CommonConverters {
  default EmailAddress toEmail(String s){ return s==null ? null : EmailAddress.of(s); }
  default Slug toSlug(String s){ return s==null ? null : Slug.of(s); }
  default NormalizedName toName(String s){ return s==null ? null : NormalizedName.of(s); }

  default String fromEmail(EmailAddress v){ return v==null? null : v.value(); }
  default String fromSlug(Slug v){ return v==null? null : v.value(); }
  default String fromName(NormalizedName v){ return v==null? null : v.value(); }
}
```

```java
// src/main/java/com/example/shared/mapstruct/GlobalMapperConfig.java
package com.example.shared.mapstruct;

import org.mapstruct.MapperConfig;
import org.mapstruct.NullValuePropertyMappingStrategy;

@MapperConfig(
  componentModel = "spring",
  uses = { CommonConverters.class },
  nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface GlobalMapperConfig {}
```

> Any mapper using `@Mapper(config = GlobalMapperConfig.class)` now auto-maps `String -> EmailAddress/Slug/NormalizedName` by type—no per-field ceremony.

---

### 6) Optional JPA converters (store VOs cleanly)

```java
// src/main/java/com/example/shared/jpa/EmailAttr.java
package com.example.shared.jpa;

import com.example.shared.vo.EmailAddress;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;

@Converter(autoApply = true)
public class EmailAttr implements AttributeConverter<EmailAddress,String> {
  public String convertToDatabaseColumn(EmailAddress v){ return v==null? null : v.value(); }
  public EmailAddress convertToEntityAttribute(String c){ return c==null? null : EmailAddress.of(c); }
}
```

(Repeat for `Slug` and `NormalizedName` if you store them in entities.)

---

### 7) DB guard (Flyway example)

```sql
-- src/main/resources/db/migration/V1__category_indexes.sql
-- Case-insensitive unique "name" (PostgreSQL)
CREATE UNIQUE INDEX uq_category_name_ci ON category (lower(name));

-- Align column sizes with DTO/VO limits (example)
-- ALTER TABLE category ALTER COLUMN name TYPE varchar(64);
```

---

### 8) Maven/Gradle bits (if you use MapStruct)

**Maven:**

```xml
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.6.2</version>
</dependency>
<annotationProcessorPaths>
  <path>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.6.2</version>
  </path>
</annotationProcessorPaths>
```

**Gradle (Kotlin DSL):**

```kotlin
implementation("org.mapstruct:mapstruct:1.6.2")
annotationProcessor("org.mapstruct:mapstruct-processor:1.6.2")
```

You already have Spring Boot + Jakarta Validation in dependencies for `@Valid` and ProblemDetail.

---

## How to use (today)

* Keep building with DTOs + `@Valid`.
* Sensitive fields in DTOs → annotate with `@Raw`.
* Services can stay simple; strings in `@RequestBody` are already cleaned.
* When a field’s correctness matters everywhere, introduce a VO and change the service signature to accept that VO (or use MapStruct to create it).

This kit keeps your edge clean, your errors consistent, and your core ready to harden—without ceremony creep.
