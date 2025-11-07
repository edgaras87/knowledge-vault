---
title: Quick Starter
date: 2025-11-03
tags: 
summary: 
aliases:
  - Spring Boot Properties Quick Starter Cheatsheet

---

# Spring Boot Properties — Cheatsheet

## 0) Where configs live

* `src/main/resources/application.yml` (or `.properties`) — base defaults for all profiles.
* `application-<profile>.yml` — profile-specific overrides (e.g., `application-dev.yml`).
* External files/env/CLI can override everything (see precedence).

```yaml
# application.yml
spring:
  application:
    name: shopping-cart
server:
  port: 8080

app:
  host: "https://test.local"
  timeout: 10s
  features:
    signup: true
  admins:
    - "alice@example.com"
    - "bob@example.com"
  limits:
    uploads: 50MB
    itemsPerPage: 20
```

## 1) Property formats + types

* **Relaxed binding**: `app.host`, `app-host`, `APP_HOST` → all map to `app.host`.
* Types auto-convert:
  `Duration` (e.g., `10s`, `2m`, `1h`), `DataSize` (e.g., `50MB`), `URI`, `URL`, `Path`, enums, etc.
* Lists & maps:
  YAML is friendlier; in `.properties` use indexes or dotted keys.

```properties
# application.properties equivalents
app.host=https://test.local
app.timeout=10s
app.features.signup=true
app.admins[0]=alice@example.com
app.admins[1]=bob@example.com
app.limits.uploads=50MB
app.limits.items-per-page=20
```

## 2) Reading values (quick & strong)

### a) Quick pick: `@Value` (fine for one-offs, but avoid for bulk)

```java
@Value("${app.host}") URI host;
@Value("${app.timeout:5s}") Duration timeout; // with default
// Placeholders inside values:
@Value("${external.base}/v1/users") String usersEndpoint;
```

### b) Strongly-typed binding: `@ConfigurationProperties` (preferred)

* Put in `com.edge.shopping_cart.config` (or `…/shared/config`).
* Works with classes **or records**; constructor binding is automatic in Spring Boot 3+.

```java
// src/main/java/com/edge/shopping_cart/config/AppProps.java
package com.edge.shopping_cart.config;

import java.net.URI;
import java.time.Duration;
import java.util.List;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "app")
public record AppProps(
  URI host,
  Duration timeout,
  Features features,
  List<String> admins,
  Limits limits
) {
  public record Features(boolean signup) {}
  public record Limits(org.springframework.util.unit.DataSize uploads, int itemsPerPage) {}
}
```

!!! note
    Here `Features` and `Limits` are nested records to group related settings.

Think of it as a tiny, immutable type namespaced under AppProps for clarity—not an inner, non-static class.

Why this works well:

- Structure mirrors config. app.features.signup maps to AppProps.Features.signup.
- No outer-instance overhead. Static-by-default member records are lean.
- Encapsulation. If Features/Limits only make sense inside AppProps, keep them there. Make them public or private depending on how widely they should be used.

!!! tip
    When not to nest: if Features or Limits are reused across modules/packages, promote them to top-level records to avoid awkward imports like AppProps.Features.

Enable scanning once (either way works):

```java
// Option A: annotate your main app class
@SpringBootApplication
@EnableConfigurationProperties(AppProps.class) // ({ AppProps.class})
public class Application { /* ... */ }

// Option B: component-scan for all @ConfigurationProperties classes
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { /* ... */ }
```

Use it anywhere (DI like any bean):

```java
@Service
public class LinkBuilder {
  private final AppProps props;
  public LinkBuilder(AppProps props) { this.props = props; }

  public URI problemsBase() { return props.host().resolve("/problems/"); }
}
```

### c) Validation for configs

```java
// Add @Validated on class, then javax/jakarta validation on fields
import jakarta.validation.constraints.*;
import org.springframework.validation.annotation.Validated;

@ConfigurationProperties(prefix = "mail")
@Validated
public record MailProps(
  @NotBlank String from,
  @Positive int retries
) {}
```

If validation fails at startup, the app refuses to boot—good.

## 3) Profiles (dev, test, prod)

* Name files `application-dev.yml`, `application-prod.yml`.
* Activate:

  * Env: `SPRING_PROFILES_ACTIVE=dev`
  * CLI: `--spring.profiles.active=dev`
  * Maven/Gradle run: `-Dspring.profiles.active=dev`
* Profile groups:

```yaml
# application.yml
spring:
  profiles:
    group:
      local: [dev, debug]
```

### Example: dev vs prod behavior toggles

Now that you can bend YAML to your will, the next step is **teaching your application to *behave differently depending on where it runs***—without touching code.

This is the moment where you stop thinking “project” and start thinking **system**.

Moving from:

> “I hard-code values and tweak code to test stuff”

to:

> “I shape environments — my app adapts.”

Welcome to profiles, environment-based settings, and tiny real-world config tricks. You’re basically leveling up from “developer” to “small-scale DevOps monk”.

---

### Step 1: Create `dev` vs `prod` profiles

Two files:

**`application.yml`**

```yaml
spring:
  application:
    name: shop

app:
  host: "http://localhost:8080"
  features:
    signup: true
```

**`application-prod.yml`**

```yaml
app:
  host: "https://myrealapp.com"
  features:
    signup: false # maybe closed beta in production
```

Run dev mode (default):

```
./mvnw spring-boot:run
```

Run “pretend real world”:

```
./mvnw spring-boot:run -Dspring.profiles.active=prod
```

You just changed your universe with one flag.

This is what engineers crave: **behavior toggles without code edits**.

---

### Step 2: Use config *inside logic*, not just for decoration

Imagine feature flagging a controller path:

```java
@RestController
class SignupController {
  private final AppProps cfg;

  SignupController(AppProps cfg) { this.cfg = cfg; }

  @PostMapping("/signup")
  public String signup() {
    if (!cfg.features().signup()) {
      throw new IllegalStateException("Signup disabled");
    }
    return "Signup OK";
  }
}
```

Now profiles literally control features. Not tech fluff — **business rules**.

This is where config stops being mechanics and becomes architecture.


## 4) Overriding & precedence (highest → lowest)

1. **Command line args**: `--app.host=https://live.local`
2. **OS env vars**: `APP_HOST=https://live.local`
3. **`application-<profile>.yml`** of the active profile
4. **`application.yml`** (packaged)
5. **Spring defaults**

Handy: ENV → property mapping uses upper-case + underscores:

* `APP_LIMITS_UPLOADS=100MB` → `app.limits.uploads`
* `SPRING_DATASOURCE_URL=jdbc:postgresql://…`

## 5) Config data import & extra locations

Add more files or make them optional:

```yaml
# application.yml
spring:
  config:
    import: optional:file:.env[.properties],optional:file:./secrets.yml
```

Also:

* `--spring.config.additional-location=./config/` to bolt on another folder.
* Spring Cloud Config server uses the same “config data” mechanism (`spring.config.import=optional:configserver:`), if you adopt it later.


### Example: `.env.local` for developer overrides

Create `.env.local` in project root:

```
APP_HOST=http://localhost:9999
APP_FEATURES_SIGNUP=false
```

And import it:

```yaml
spring:
  config:
    import: optional:file:.env.local
```

You just learned the pattern that scales all the way to **Kubernetes Secrets**, **CI pipelines**, and **Docker deployments**.

No big deal — just touching the machinery used by billion-dollar systems.

## 6) Placeholders, defaults, and SpEL

* Placeholders with defaults: `${prop.key:defaultValue}`
* You can reference other properties: `cdn.url: "${app.host}/static"`
* SpEL works **only** in `@Value`, not in `application.yml`. Keep SpEL rare.



### Introduce a “defaults + override” mindset

Pretend you need pagination default. Instead of sprinkling magic numbers:

```java
int limit = 20;
```

You bind it:

```yaml
app:
  limits:
    itemsPerPage: 20
```

In code:

```java
public int defaultPageSize() {
  return cfg.limits().itemsPerPage();
}
```

Later someone says “change to 50?”
You don't chase code — you flip YAML.
Clean systems live longer.


## 7) Typical property sets you’ll use

```yaml
# HTTP & compression
server:
  port: 8080
  compression:
    enabled: true
    min-response-size: 2KB

# Datasource
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app
    username: app
    password: app
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  sql:
    init:
      mode: never

# Logging
logging:
  level:
    root: INFO
    com.edge.shopping_cart: DEBUG
  pattern:
    level: "%5p"

# Actuator (if used)
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,loggers
  endpoint:
    health:
      show-details: when_authorized
```

## 8) Testing with properties

* One-off overrides on a test:

```java
@SpringBootTest(properties = {
  "app.features.signup=false",
  "app.timeout=1s"
})
class MyTest { }
```

* With a test profile:

  * Create `src/test/resources/application-test.yml`
  * Activate: `@ActiveProfiles("test")`
* Dynamic values from Testcontainers, etc.:

```java
@Testcontainers
@SpringBootTest
class DbTest {
  @Container static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");

  @DynamicPropertySource
  static void dbProps(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", db::getJdbcUrl);
    r.add("spring.datasource.username", db::getUsername);
    r.add("spring.datasource.password", db::getPassword);
  }
}
```

## 9) Secrets & production hygiene

* Don’t commit passwords. Use env vars, `.docker.env`, Kubernetes Secrets, or external files via `spring.config.import`.
* Prefer **URIs** for external service endpoints; avoid string concatenation.
* Keep a clear **contract** for your `@ConfigurationProperties` classes; that’s your app’s “config API”.

## 10) Common pitfalls & fixes

* **“Value not bound”**: prefix mismatch or wrong relaxed name. Check `app.host` vs `appHost`.
* **Validation isn’t running**: add `@Validated` on the config class.
* **List binding fails in `.properties`**: use indexed keys or switch to YAML.
* **`@Value` everywhere**: migrate to `@ConfigurationProperties` to avoid stringly-typed config soup.
* **Profile file not picked**: verify `SPRING_PROFILES_ACTIVE` (no spaces, commas allowed: `dev,debug`).

---

### Minimal starter you can drop in today

**`src/main/java/com/edge/shopping_cart/config/AppProps.java`**

```java
package com.edge.shopping_cart.config;

import java.net.URI;
import java.time.Duration;
import java.util.List;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("app")
public record AppProps(
  URI host,
  Duration timeout,
  List<String> admins
) { }
```

**`src/main/java/com/edge/shopping_cart/Application.java`**

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProps.class)
public class Application {
  public static void main(String[] args) { SpringApplication.run(Application.class, args); }
}
```

**`src/main/resources/application.yml`**

```yaml
app:
  host: "https://test.local"
  timeout: 10s
  admins: [ "admin@example.com" ]
```

**Use it**

```java
@RestController
class HomeController {
  private final AppProps cfg;
  HomeController(AppProps cfg) { this.cfg = cfg; }

  @GetMapping("/")
  String home() {
    return "Hello from Shopping Cart — problems at: " + cfg.host().resolve("/problems/");
  }
}
```


## Next Moves (in order)

---

## 11) Bind + validate more than “hello world”

Add one more real config object and fail-fast if it’s wrong.

```java
// src/main/java/com/edge/shopping_cart/config/MailProps.java
package com.edge.shopping_cart.config;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

@ConfigurationProperties("mail")
@Validated
public record MailProps(
  @NotBlank String from,
  @Email String support,
  @PositiveOrZero int retries
) {}
```

```java
// Application.java
@SpringBootApplication
@EnableConfigurationProperties({ AppProps.class, MailProps.class })
public class Application { … }
```

```yaml
# application.yml
mail:
  from: "noreply@test.local"
  support: "support@test.local"
  retries: 2
```

This makes config an API: clear contract, startup validation, no surprises in prod.

---

## 12) Give Problem Details a stable base from config

You asked about where `/problems/*` should live. Drive it with a property.

```yaml
# application.yml
problems:
  base: "https://test.local/problems/"
```

```java
// src/main/java/com/edge/shopping_cart/config/ProblemProps.java
@ConfigurationProperties("problems")
public record ProblemProps(URI base) {}
```

```java
// src/main/java/com/edge/shopping_cart/web/ProblemLinks.java
@Component
public class ProblemLinks {
  private final ProblemProps props;
  public ProblemLinks(ProblemProps props) { this.props = props; }
  public URI type(String slug) { return props.base().resolve(slug); } // e.g., "duplicate-category"
}
```

```java
// src/main/java/com/edge/shopping_cart/web/GlobalExceptionHandler.java
@RestControllerAdvice
@RequiredArgsConstructor
class GlobalExceptionHandler {
  private final ProblemLinks links;

  @ExceptionHandler(DuplicateCategory.class)
  ResponseEntity<ProblemDetail> handle(DuplicateCategory ex, HttpServletRequest req) {
    var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
    pd.setType(links.type("duplicate-category"));
    pd.setTitle("Duplicate category");
    pd.setDetail(ex.getMessage());
    pd.setInstance(URI.create(req.getRequestURI()));
    return ResponseEntity.status(HttpStatus.CONFLICT).body(pd);
  }
}
```

Why a full absolute URL? It’s a stable, dereferenceable spec anchor you can document (even if it 404s during practice). Later, serve static docs at that path.

---

## 13) Property-driven beans (let config actually change behavior)

Show a real toggle.

```yaml
app:
  cors:
    enabled: true
    origins: ["http://localhost:4200"]
```

```java
@ConfigurationProperties("app.cors")
public record CorsProps(boolean enabled, List<String> origins) {}

@Configuration
@EnableConfigurationProperties(CorsProps.class)
class WebConfig {
  @Bean
  @ConditionalOnProperty(prefix = "app.cors", name = "enabled", havingValue = "true")
  CorsFilter cors(CorsProps cfg) {
    var src = new UrlBasedCorsConfigurationSource();
    var c = new CorsConfiguration();
    c.setAllowedOrigins(cfg.origins());
    c.addAllowedHeader("*");
    c.addAllowedMethod("*");
    src.registerCorsConfiguration("/**", c);
    return new CorsFilter(src);
  }
}
```

Now you can flip CORS per environment without touching code.

---

## 14) Profiles that actually differ

Split dev vs prod cleanly.

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app
logging.level.com.edge.shopping_cart: DEBUG
app:
  host: "http://localhost:8080"

# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/app
logging.level.com.edge.shopping_cart: INFO
app:
  host: "https://api.example.com"
```

Run:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
java -jar app.jar --spring.profiles.active=prod
```

---

## 15) Environment wiring you’ll actually use

Three ways you’ll override in practice:

**CLI**

```bash
java -jar app.jar --problems.base=https://api.example.com/problems/
```

**ENV (Docker/K8s/systemd)**

```bash
export PROBLEMS_BASE=https://api.example.com/problems/
```

**Additional files**

```yaml
spring:
  config:
    import: optional:file:./secrets.yml
```

---

## 16) Testing patterns with properties (fast feedback)

* **Per-test overrides**

```java
@SpringBootTest(properties = "problems.base=https://test.local/problems/")
class AnyTest { }
```

* **Test profile**

```
src/test/resources/application-test.yml
```

```java
@ActiveProfiles("test")
class RepoTest { }
```

* **Dynamic from Testcontainers**

```java
@DynamicPropertySource
static void db(DynamicPropertyRegistry r) {
  r.add("spring.datasource.url", container::getJdbcUrl);
}
```

---

## 17) Replace scattered `@Value` with typed configs

Migration recipe:

1. Grep for `@Value("...app.`
2. Create a single `@ConfigurationProperties("app")` record that covers them.
3. Inject that record where needed.
4. Delete magic strings one by one.
   Outcome: fewer typos, better IDE help, easier diffing in PRs.

---

## 18) Ops hygiene: see what you actually run with

Expose minimal Actuator and query the live env safely.

```yaml
management:
  endpoints.web.exposure.include: env,health,info
  endpoint.env.enabled: true
```

Then:

```
GET /actuator/env/app.host
GET /actuator/env/PROBLEMS_BASE
```

This is your “why isn’t my property taking” lie detector.

---

## 19) Vault drop-ins (to keep you organized)

```
cheatsheets/properties/
├─ index.md                  # overview + precedence
├─ binding.md                # @ConfigurationProperties patterns + validation
├─ profiles.md               # dev/test/prod, groups, activation
├─ env-and-secrets.md        # Docker, systemd, k8s, config import
├─ testing.md                # @SpringBootTest props, DynamicPropertySource
├─ problem-links.md          # problems.base pattern + URI builder helper
└─ property-driven-beans.md  # @ConditionalOnProperty, toggles (CORS, Jackson, etc.)
```

Each page: short rationale, 1–2 “golden” snippets, pitfalls.

---

## 20) Two tiny “do now” tasks

* **Add `problems.base`** and wire `ProblemLinks` as above. You’ll feel the benefit immediately when you add more errors.
* **Flip a real toggle** (`app.cors.enabled`) and confirm behavior changes between `dev` and `prod`.

