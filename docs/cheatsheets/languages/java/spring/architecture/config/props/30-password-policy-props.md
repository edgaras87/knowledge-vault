---
title: PasswordPolicyProps
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Centralize password policy settings into a single validated record with rich types, enabling services to enforce complexity, blacklist, breach checks, reuse limits, and expiry without string parsing.
aliases:
    - Spring Config Props Layer - PasswordPolicyProps Cheatsheet
---

# PasswordPolicyProps — typed password policy

Centralize **complexity rules**, **blacklists**, **breach checks**, **reuse limits**, and **expiry** in one validated record. Services get **typed** values; no string parsing.

---

## YAML we’re binding

```yaml
security:
  password:
    min-length: 12
    max-length: 128

    require:
      upper: 1        # at least N uppercase letters
      lower: 1        # at least N lowercase letters
      digit: 1        # at least N digits
      symbol: 1       # at least N symbols (from allowed-symbols)

    allowed-symbols: "!@#$%^&*()_+-=[]{};:,.?/\\|"
    forbid-whitespace: true
    forbid-username:  true
    forbid-email:     true

    breached-check:
      enabled: true
      provider: hibp   # hibp (k-anon) | none (custom later)
      min-score: 1     # 0..4, how strict you are with provider’s scoring

    dictionary:
      enabled: true
      source: "classpath:security/common-passwords.txt"  # one-per-line

    reuse:
      history-size: 5   # keep last N password hashes per user
      min-delta: 3      # require distance from last N by some metric (levenshtein/entropy)

    expiry:
      enabled: false
      ttl: 365d         # rotate if you *must* (many orgs keep this off)
```

---

## Where it lives

* **Class:** `com.example.app.config.props.PasswordPolicyProps`
* **Used by:** registration/reset flows, password change use cases, validators

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/props/config/PasswordPolicyProps.java
package com.example.app.config.props;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.core.io.Resource;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;

@Validated
@ConfigurationProperties(prefix = "security.password")
public record PasswordPolicyProps(
    @Min(4)  @Max(1024) int minLength,
    @Min(4)  @Max(1024) int maxLength,

    @Valid @NotNull Require require,
    @NotNull Boolean forbidWhitespace,
    @NotNull Boolean forbidUsername,
    @NotNull Boolean forbidEmail,

    @NotBlank String allowedSymbols,

    @Valid @NotNull BreachedCheck breachedCheck,
    @Valid @NotNull Dictionary dictionary,
    @Valid @NotNull Reuse reuse,
    @Valid @NotNull Expiry expiry
) {
  public PasswordPolicyProps {
    if (minLength > maxLength) {
      throw new IllegalArgumentException("security.password.min-length must be <= max-length");
    }
    if (require.upper < 0 || require.lower < 0 || require.digit < 0 || require.symbol < 0) {
      throw new IllegalArgumentException("security.password.require counts must be >= 0");
    }
    // If symbols are required, you must actually allow some symbols.
    if (require.symbol > 0 && (allowedSymbols == null || allowedSymbols.isBlank())) {
      throw new IllegalArgumentException("allowed-symbols must be non-blank when require.symbol > 0");
    }
    if (breachedCheck.enabled && breachedCheck.provider == Provider.none) {
      throw new IllegalArgumentException("breached-check.enabled=true requires a real provider");
    }
    if (reuse.historySize < 0 || reuse.minDelta < 0) {
      throw new IllegalArgumentException("reuse.history-size/min-delta must be >= 0");
    }
    if (expiry.enabled && (expiry.ttl == null || expiry.ttl.isNegative() || expiry.ttl.isZero())) {
      throw new IllegalArgumentException("expiry.ttl must be positive when expiry.enabled=true");
    }
  }

  public record Require(
      int upper,
      int lower,
      int digit,
      int symbol
  ) {}

  public record BreachedCheck(
      boolean enabled,
      @NotNull Provider provider,
      @Min(0) @Max(4) int minScore
  ) {
    public enum Provider { hibp, none } // extend later if needed
  }

  public record Dictionary(
      boolean enabled,
      Resource source // classpath: or file:; one password per line
  ) {}

  public record Reuse(
      @Min(0) int historySize,
      @Min(0) int minDelta
  ) {}

  public record Expiry(
      boolean enabled,
      Duration ttl
  ) {}
}
```

---

## Enable binding (if not scanning already)

```java
// src/main/java/com/example/app/props/Application.java
package com.example.app;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;

@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { }
```

---

## Quick consumer sketch

```java
// src/main/java/com/example/app/props/security/PasswordPolicy.java
package com.example.app.security;

import com.example.app.config.props.PasswordPolicyProps;
import org.springframework.stereotype.Component;

@Component
public class PasswordPolicy {
  private final PasswordPolicyProps p;

  public PasswordPolicy(PasswordPolicyProps p) { this.p = p; }

  public int minLength() { return p.minLength(); }
  public int maxLength() { return p.maxLength(); }

  public String allowedSymbols() { return p.allowedSymbols(); }
  public boolean forbidWhitespace() { return p.forbidWhitespace(); }
  public boolean forbidUsername() { return p.forbidUsername(); }
  public boolean forbidEmail() { return p.forbidEmail(); }

  public int reqUpper() { return p.require().upper(); }
  public int reqLower() { return p.require().lower(); }
  public int reqDigit() { return p.require().digit(); }
  public int reqSymbol() { return p.require().symbol(); }

  public boolean breachedEnabled() { return p.breachedCheck().enabled(); }
  // …and so on for reuse/expiry.
}
```

Wire this into your **register / change-password** use cases and validators.

---

## ENV equivalents

```bash
SECURITY_PASSWORD_MIN_LENGTH=12
SECURITY_PASSWORD_MAX_LENGTH=128

SECURITY_PASSWORD_REQUIRE_UPPER=1
SECURITY_PASSWORD_REQUIRE_LOWER=1
SECURITY_PASSWORD_REQUIRE_DIGIT=1
SECURITY_PASSWORD_REQUIRE_SYMBOL=1

SECURITY_PASSWORD_ALLOWED_SYMBOLS='!@#$%^&*()_+-=[]{};:,.?/\\|'
SECURITY_PASSWORD_FORBID_WHITESPACE=true
SECURITY_PASSWORD_FORBID_USERNAME=true
SECURITY_PASSWORD_FORBID_EMAIL=true

SECURITY_PASSWORD_BREACHED_CHECK_ENABLED=true
SECURITY_PASSWORD_BREACHED_CHECK_PROVIDER=hibp
SECURITY_PASSWORD_BREACHED_CHECK_MIN_SCORE=1

SECURITY_PASSWORD_DICTIONARY_ENABLED=true
SECURITY_PASSWORD_DICTIONARY_SOURCE=classpath:security/common-passwords.txt

SECURITY_PASSWORD_REUSE_HISTORY_SIZE=5
SECURITY_PASSWORD_REUSE_MIN_DELTA=3

SECURITY_PASSWORD_EXPIRY_ENABLED=false
SECURITY_PASSWORD_EXPIRY_TTL=365d
```

---

## Guardrails & practical notes

* **Min ≤ Max** and all **require* counts ≥ 0** — enforced in the constructor.
* If you require symbols, **`allowed-symbols` must not be blank**.
* **Breach checking** is on/off; choose a real provider (e.g., `hibp`) when enabled.
* **Dictionary** uses a `Resource` — supports `classpath:` or `file:`.
* Expiry is optional; many teams keep it **disabled** and rely on breach/reuse checks.
* Keep blacklist files and any breach-check API keys out of Git.

---

## Tests (binding + invariants)

```java
// src/test/java/com/example/app/props/config/PasswordPolicyPropsTest.java
package com.example.app.config.props;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PasswordPolicyPropsTest {

  private ApplicationContextRunner run(String... props) {
    return new ApplicationContextRunner()
        .withUserConfiguration(Bootstrap.class)
        .withPropertyValues(props);
  }

  @Test
  void minMustNotExceedMax() {
    assertThatThrownBy(() -> run(
        "security.password.min-length=20",
        "security.password.max-length=10",
        "security.password.require.upper=1",
        "security.password.require.lower=1",
        "security.password.require.digit=1",
        "security.password.require.symbol=0",
        "security.password.allowed-symbols=!@#",
        "security.password.forbid-whitespace=true",
        "security.password.forbid-username=true",
        "security.password.forbid-email=true",
        "security.password.breached-check.enabled=false",
        "security.password.breached-check.provider=none",
        "security.password.breached-check.min-score=0",
        "security.password.dictionary.enabled=false",
        "security.password.reuse.history-size=5",
        "security.password.reuse.min-delta=3",
        "security.password.expiry.enabled=false"
    ).run(c -> c.getBean(PasswordPolicyProps.class))).hasMessageContaining("min-length must be <= max-length");
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(PasswordPolicyProps.class)
  static class Bootstrap {}
}
```

---

## What to build next

* `SecurityFilterChain` skeleton that **reads** `PasswordPolicyProps` to configure validators.
* A small **`PasswordValidator`** service that enforces the counts, symbol set, and blacklist/breach checks.
* A CLI/test helper to **scan dictionaries** and benchmark policy changes.
