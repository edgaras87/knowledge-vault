---
title: Quick Starter  
date: 2025-11-02
tags: 
    - java
    - spring
    - validation
    - architecture
summary: A practical cheatsheet for using Jakarta Validation (formerly javax.validation) in Spring Boot 3+ applications, covering core annotations, DTO validation, method validation, custom constraints, error handling, and best practices.
aliases:
  - Jakarta Validation Cheatsheet

---

# Jakarta Validation — Practical Cheatsheet (Spring Boot 3+)

## 0) Setup

**Gradle (Kotlin DSL):**

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-validation") // Hibernate Validator
}
```

**Maven:**

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
</dependencies>
```

---

## 1) Core annotations (most used)

```java
@NotNull            // must not be null
@NotBlank           // string has non-whitespace chars
@Size(min=, max=)   // strings, collections
@Email              // RFC5322-ish email check
@Pattern(regexp=)   // regex
@Min, @Max          // long/integers
@Positive, @PositiveOrZero, @Negative
@Past, @PastOrPresent, @Future, @FutureOrPresent // dates
@Valid              // cascade validation into nested object(s)
```

---

## 2) DTOs in Controllers

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.*;

public record CreateUserRequest(
    @NotBlank @Email @Size(max = 255) String email,
    @NotBlank @Size(min = 3, max = 100) String name,
    @NotBlank @Size(min = 8, max = 72) String password
) {
  // Optional: sanitize
  public CreateUserRequest {
    email = email == null ? null : email.trim().toLowerCase();
    name = name == null ? null : name.trim();
    password = password == null ? null : password.trim();
  }
}
```

```java
@RestController
@RequestMapping("/users")
class UserController {
  private final UserService service;
  private final UserMapper mapper;

  UserController(UserService service, UserMapper mapper) {
    this.service = service; this.mapper = mapper;
  }

  @PostMapping
  public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req,
                                             UriComponentsBuilder uriBuilder) {
    var created = service.create(mapper.toCommand(req));
    var location = uriBuilder.path("/users/{id}").buildAndExpand(created.id()).toUri();
    return ResponseEntity.created(location).body(mapper.toResponse(created));
  }
}
```

**Nested & collections:**

```java
public record Address(@NotBlank String street, @NotBlank String city) {}

public record OrderRequest(
    @NotNull Long userId,
    @Valid Address shipping,                   // cascade
    @NotEmpty @Valid java.util.List<@NotNull LineItem> items // elements validated
) {}

public record LineItem(@NotBlank String sku, @Positive int qty) {}
```

---

## 3) Method validation on any Spring bean

Enable by putting `@Validated` on the **class** (or method). Works for services, schedulers, listeners.

```java
import org.springframework.validation.annotation.Validated;

@Validated
@Service
class UserService {

  public User create(@Valid CreateUserCommand cmd) { /* ... */ }

  public User getById(@Positive long id) { /* ... */ }

  public @Valid UserResponse findByEmail(@NotBlank @Email String email) { /* ... */ }
}
```

**Programmatic validation when needed:**

```java
@Service
class SomeAssembler {
  private final jakarta.validation.Validator validator;

  SomeAssembler(Validator validator) { this.validator = validator; }

  void ensureValid(Object obj) {
    var violations = validator.validate(obj);
    if (!violations.isEmpty()) throw new IllegalArgumentException(violations.toString());
  }
}
```

---

## 4) Validation Groups (Create vs Update)

```java
public interface Create {}
public interface Update {}

public record UserUpsert(
    @NotBlank(groups = Create.class) String name,       // required on create
    @Email(groups = {Create.class, Update.class}) String email
) {}

@Validated(Create.class)
@PostMapping("/users") void create(@RequestBody @Valid UserUpsert body) {}

@Validated(Update.class)
@PatchMapping("/users/{id}") void update(@RequestBody @Valid UserUpsert body) {}
```

---

## 5) Custom constraint (field-level) — example: strong password

**Annotation:**

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
public @interface StrongPassword {
  String message() default "Weak password";
  Class<?>[] groups() default {};
  Class<? extends jakarta.validation.Payload>[] payload() default {};
}
```

**Validator:**

```java
public class StrongPasswordValidator
        implements jakarta.validation.ConstraintValidator<StrongPassword, String> {
  @Override public boolean isValid(String v, ConstraintValidatorContext ctx) {
    if (v == null) return false;
    return v.length() >= 8 && v.matches(".*[A-Z].*") && v.matches(".*\\d.*");
  }
}
```

**Use:**

```java
public record Registration(@StrongPassword String password) {}
```

---

## 6) Constraint composition (bundle common rules)

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {})
@NotBlank @Email @Size(max = 255)
public @interface ValidEmail {
  String message() default "Invalid email";
  Class<?>[] groups() default {};
  Class<? extends jakarta.validation.Payload>[] payload() default {};
}
```

---

## 7) Class-level constraint (cross-field)

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {
  String message() default "end must be after start";
  Class<?>[] groups() default {};
  Class<? extends jakarta.validation.Payload>[] payload() default {};
}

public record Booking(java.time.LocalDate start, java.time.LocalDate end) {}

public class DateRangeValidator
        implements jakarta.validation.ConstraintValidator<ValidDateRange, Booking> {
  @Override public boolean isValid(Booking b, ConstraintValidatorContext c) {
    if (b == null) return true;
    if (b.start() == null || b.end() == null) return true; // let @NotNull handle if needed
    return b.end().isAfter(b.start());
  }
}
```

---

## 8) Global error handling → Problem Details (Spring 6+)

```java
@RestControllerAdvice
class GlobalExceptionHandler {

  // Bean validation on @RequestBody, @RequestParam, etc.
  @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException.class)
  ResponseEntity<ProblemDetail> handleInvalid(MethodArgumentNotValidException ex) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    var errors = new java.util.LinkedHashMap<String,Object>();
    ex.getBindingResult().getFieldErrors().forEach(fe ->
        errors.put(fe.getField(), fe.getDefaultMessage()));
    pd.setProperty("errors", errors);
    pd.setType(URI.create("https://api.example.com/problems/validation-error"));
    return ResponseEntity.badRequest().body(pd);
  }

  // Method validation on @Validated beans (service params/returns)
  @ExceptionHandler(jakarta.validation.ConstraintViolationException.class)
  ResponseEntity<ProblemDetail> handleViolations(jakarta.validation.ConstraintViolationException ex) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Constraint violation");
    var errors = new java.util.LinkedHashMap<String, String>();
    ex.getConstraintViolations().forEach(v -> errors.put(v.getPropertyPath().toString(), v.getMessage()));
    pd.setProperty("errors", errors);
    pd.setType(URI.create("https://api.example.com/problems/constraint-violation"));
    return ResponseEntity.badRequest().body(pd);
  }

  // DB unique constraint → translate cleanly
  @ExceptionHandler(org.springframework.dao.DataIntegrityViolationException.class)
  ResponseEntity<ProblemDetail> handleIntegrity(org.springframework.dao.DataIntegrityViolationException ex) {
    var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setTitle("Integrity constraint violation");
    pd.setDetail("Duplicate or invalid relational data.");
    pd.setType(URI.create("https://api.example.com/problems/integrity-violation"));
    return ResponseEntity.status(HttpStatus.CONFLICT).body(pd);
  }
}
```

---

## 9) Kafka listener (or any message listener) with validation

```java
@Component
@Validated // enables method parameter validation
class UserEventsListener {

  @KafkaListener(topics = "users.create")
  public void handleSingle(@Valid UserCreatedEvent event) {
    // safe to use
  }

  @KafkaListener(topics = "users.create.batch")
  public void handleBatch(@Valid java.util.List<@Valid UserCreatedEvent> events) {
    // each element validated
  }
}

public record UserCreatedEvent(
    @ValidEmail String email,
    @NotBlank String name
) {}
```

---

## 10) Uniqueness strategy (don’t query DB inside validators)

**Service (fast pre-check + rely on DB unique index):**

```java
@Service
class UserService {
  private final UserRepository repo;

  UserService(UserRepository repo) { this.repo = repo; }

  public User create(@Valid CreateUserCommand cmd) {
    if (repo.existsByEmail(cmd.email())) {
      throw new DuplicateEmailException(cmd.email()); // friendly fail-fast
    }
    try {
      return repo.save(new User(cmd.email(), cmd.name(), cmd.passwordHash()));
    } catch (org.springframework.dao.DataIntegrityViolationException e) {
      // Race or other constraint → translate cleanly
      throw new DuplicateEmailException(cmd.email());
    }
  }
}
```

**DB:**

```sql
CREATE UNIQUE INDEX ux_users_email ON users(email);
```

---

## 11) i18n messages (optional)

**`src/main/resources/messages.properties`:**

```
user.email.invalid=Invalid email address
user.name.required=Name is required
```

**Use:**

```java
public record Sample(
  @NotBlank(message = "{user.name.required}") String name,
  @Email(message = "{user.email.invalid}") String email
) {}
```

---

## 12) Testing snippets

**JUnit: controller validation (slice):**

```java
@WebMvcTest(controllers = UserController.class)
class UserControllerTest {
  @Autowired MockMvc mvc;

  @Test void rejectsBlankEmail() throws Exception {
    var json = """
      {"email":"  ","name":"Ed","password":"Secret123"}
      """;
    mvc.perform(post("/users").contentType(MediaType.APPLICATION_JSON).content(json))
       .andExpect(status().isBadRequest())
       .andExpect(jsonPath("$.errors.email").exists());
  }
}
```

**JUnit: method validation in services:**

```java
@SpringBootTest
class UserServiceValidationTest {
  @Autowired UserService service;

  @Test
  void rejectsInvalidId() {
    assertThrows(jakarta.validation.ConstraintViolationException.class,
        () -> service.getById(0)); // @Positive long id
  }
}
```

**Plain validator unit test:**

```java
class StrongPasswordValidatorTest {
  private final jakarta.validation.Validator validator =
      jakarta.validation.Validation.buildDefaultValidatorFactory().getValidator();

  record Pwd(@StrongPassword String p) {}
  @Test void weakPasswordRejected() {
    var v = validator.validate(new Pwd("abc"));
    assertFalse(v.isEmpty());
  }
}
```

---

## 13) Suggested folders (fits `web/ app/ domain/ infra/`)

```
src/main/java/com/example/
  web/
    user/
      UserController.java
      dto/
        CreateUserRequest.java
        UserResponse.java
  app/
    user/
      UserService.java     // @Validated here
      command/
        CreateUserCommand.java
  domain/
    user/
      User.java
      DuplicateEmailException.java
  infra/
    user/
      UserRepository.java
  common/validation/
    annotation/
      StrongPassword.java
      ValidEmail.java
      ValidDateRange.java
    validator/
      StrongPasswordValidator.java
      DateRangeValidator.java
  common/error/
    GlobalExceptionHandler.java
```

---

## 14) Quick rules to remember

* Put **`@Valid` on controller DTOs** and nested fields/collections.
* Put **`@Validated` on beans** whose method parameters/returns you want validated (services, listeners).
* Use **groups** for `Create` vs `Update`.
* Keep **validators pure** (no repositories). Do uniqueness with **service pre-check + DB unique index**.
* Return **Problem Details** for consistent errors.
* Prefer **channel-specific DTOs** (HTTP vs Kafka) and map to **commands** for services.

---

### Minimal starter pack (drop-in)

If you only copy three things today, copy:

1. `GlobalExceptionHandler` (ProblemDetail),
2. `@Validated` on services with one parameter/return example,
3. one custom constraint (like `ValidEmail` or `StrongPassword`).

That gives you clean failures, safe inputs, and a pattern you can extend everywhere.
