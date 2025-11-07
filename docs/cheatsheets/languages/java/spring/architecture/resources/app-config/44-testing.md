---
title: Testing 
date: 2025-11-07
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for testing configuration properties in Spring Boot applications, covering techniques for overriding, validating, and isolating configuration during tests.
aliases:
  - Spring Properties - Testing Properties Cheatsheet
---

# Testing Properties — Cheatsheet

Testing configuration ensures your app starts with correct settings, binds values predictably, and fails fast when misconfigured.  
Spring Boot offers multiple ways to inject test-only configuration: inline overrides, test profiles, dynamic values, and isolated context runners.

---

# 1. Test-specific overrides (inline)

Use `@TestPropertySource`:

```java
@SpringBootTest
@TestPropertySource(properties = {
  "app.timeout=1s",
  "app.feature.signup=false"
})
class AppPropsTest {

  @Autowired AppProps props;

  @Test
  void binds() {
    assertEquals(Duration.ofSeconds(1), props.timeout());
    assertFalse(props.feature().signup());
  }
}
```

These override:

- YAML values  
- profile values  
- env vars  

Perfect for small unit-like tests with a Spring context.

---

# 2. Activating test profiles

Use a dedicated test profile file:

```
src/test/resources/application-test.yml
```

Activate it:

```java
@SpringBootTest
@ActiveProfiles("test")
class UserServiceTest {}
```

This loads:

```
application.yml  
application-test.yml  
```

Useful for:

- Test-only database config  
- Feature flags for testing  
- Mock service endpoints  

---

# 3. Dynamic properties (DynamicPropertySource)

Used when tests need runtime-generated values (e.g., Testcontainers).

```java
@Testcontainers
@SpringBootTest
class DbTest {

  @Container
  static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");

  @DynamicPropertySource
  static void registerProps(DynamicPropertyRegistry reg) {
    reg.add("spring.datasource.url", db::getJdbcUrl);
    reg.add("spring.datasource.username", db::getUsername);
    reg.add("spring.datasource.password", db::getPassword);
  }
}
```

Best for:

- Ephemeral DBs  
- Random ports  
- External services  

---

# 4. Isolated binding tests (ApplicationContextRunner)

When you only want to test configuration binding — not the whole app.

```java
class AppPropsRunnerTest {
  private final ApplicationContextRunner runner =
    new ApplicationContextRunner()
      .withPropertyValues(
        "app.timeout=5s",
        "app.feature.signup=true"
      )
      .withUserConfiguration(TestConfig.class);

  @Test
  void bindsCorrectly() {
    runner.run(ctx -> {
      AppProps props = ctx.getBean(AppProps.class);
      assertEquals(Duration.ofSeconds(5), props.timeout());
      assertTrue(props.feature().signup());
    });
  }

  @Configuration
  @EnableConfigurationProperties(AppProps.class)
  static class TestConfig {}
}
```

Advantages:

- Very fast  
- No web layer  
- No entire Spring Boot context  

Use this for tight, targeted config tests.

---

# 5. Testing validation (startup failures)

If a property fails validation, test should confirm startup fails:

```java
@SpringBootTest
@TestPropertySource(properties = "app.limits.itemsPerPage=0") // invalid
class InvalidPropsTest {

  @Test
  void contextFails() {
    assertThrows(Exception.class, () -> { /* context starts */ });
  }
}
```

Validation errors are part of application correctness.

---

# 6. Testing profile-specific logic

Example:

```java
@SpringBootTest
@ActiveProfiles("dev")
class DevProfileTest {

  @Autowired AppProps props;

  @Test
  void devOverridesApplied() {
    assertEquals("http://localhost:8081", props.host().toString());
  }
}
```

To prove prod behaves differently:

```java
@SpringBootTest
@ActiveProfiles("prod")
class ProdProfileTest {}
```

Useful for:

- environment-dependent limits  
- host URLs  
- feature toggles  

---

# 7. Testing secrets

Use environment variables during tests:

```bash
JWT_SECRET=test ./mvnw test
```

Or override in test itself:

```java
@TestPropertySource(properties = "jwt.secret=test-secret")
```

Never use real secrets.

---

# 8. Testing endpoint exposing config

Your `/api/app` endpoint should return shaped values, not raw config.

Test:

```java
@WebMvcTest(AppInfoController.class)
class AppInfoControllerTest {

  @Autowired MockMvc mvc;

  @Test
  void returnsAppInfo() throws Exception {
    mvc.perform(get("/api/app"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.itemsPerPage").value(25));
  }
}
```

If controller depends on `AppProps`, use:

```java
@Import(AppProps.class)
@TestPropertySource(properties = {
  "app.host=http://test",
  "app.limits.itemsPerPage=25"
})
```

---

# 9. Testing conditional beans (feature flags)

If a bean is guarded by:

```java
@ConditionalOnProperty(prefix="app.feature", name="metrics", havingValue="true")
```

Test both states:

```java
@SpringBootTest
@TestPropertySource(properties = "app.feature.metrics=true")
class MetricsEnabledTest {
  @Autowired MetricsCollector collector;  // exists
}

@SpringBootTest
@TestPropertySource(properties = "app.feature.metrics=false")
class MetricsDisabledTest {
  @Test
  void collectorMissing() {}
}
```

Check absence:

```java
assertFalse(ctx.containsBean("metricsCollector"));
```

---

# 10. Testing config behavior in slices (WebMvcTest, DataJpaTest, etc.)

Slices don’t load full config.  
Include configuration classes manually:

```java
@WebMvcTest(CategoryController.class)
@Import(AppProps.class)
@TestPropertySource(properties = {
  "app.host=http://mock",
  "app.limits.itemsPerPage=10"
})
class CategoryControllerTest {}
```

Without `@Import(AppProps.class)`, the test will fail to bind properties.

---

# 11. DO / DON'T / pitfalls

**DO**

- Use inline properties when changing a single value.  
- Use profiles for environment-level behavior.  
- Use ApplicationContextRunner for binding-only tests.  
- Use DynamicPropertySource for containers.  
- Fail fast when configuration is invalid.

**DON'T**

- Don’t use full Spring context when you only need binding.  
- Don’t load real secrets in tests.  
- Don’t mix too many override mechanisms in one test.  
- Don’t rely on implicit defaults.

**Pitfalls**

- Missing `@EnableConfigurationProperties` when testing in isolation.  
- `@WebMvcTest` not loading config classes by default.  
- Duplicate overrides hiding real behavior.  
- YAML in `src/test/resources` silently overriding expected values.

---

# 12. Quick glossary

- **@TestPropertySource** — inject inline overrides.  
- **@ActiveProfiles** — activate a test profile.  
- **DynamicPropertySource** — dynamic runtime value injection.  
- **ApplicationContextRunner** — fast, isolated config binding tests.  
- **Slice Test** — partial, focused Spring testing.

