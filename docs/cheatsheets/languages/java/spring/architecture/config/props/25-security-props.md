---
title: SecurityProps
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Bind all security-related settings into a single validated record with rich types, ensuring callers receive properly parsed configuration values.
aliases:
    - Spring Config Props Layer - SecurityProps Cheatsheet
---


# SecurityProps — typed security configuration (JWT-centric)

Bind **all security knobs** into one validated record: algorithms, key material, TTLs, audiences, and allowed clock skew. No string parsing in services; consumers get **typed** values.

---

## YAML we’re binding

```yaml
security:
  jwt:
    issuer: "com.example.app"
    audience: "shoppingcart-clients"
    algorithm: RS256           # HS256 | RS256 | ES256
    clock-skew: 30s            # allowed drift for verification

    access-token:
      ttl: 15m
    refresh-token:
      ttl: 30d

    keys:
      # For RS/ES algorithms (asymmetric):
      public: "classpath:keys/jwt.pub.pem"
      private: "file:/etc/keys/jwt.key.pem"
      # For HS algorithms (symmetric):
      hmac-secret: ""          # prefer env var; leave empty in yml
```

**Notes**

* Use **either** `public/private` **or** `hmac-secret`, depending on `algorithm`.
* Resource strings accept `classpath:` and `file:` URIs.

---

## Where it lives

* /props**Class:** `com.example.app.config.props.SecurityProps`
* **Used by:** `JwtKeyProvider`, `JwtService` (sign/verify), auth filters

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/props/SecurityProps.java
package com.example.app.config.props;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.Duration;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.core.io.Resource;
import org.springframework.validation.annotation.Validated;

@Validated
@ConfigurationProperties(prefix = "security.jwt")
public record SecurityProps(
    @NotBlank String issuer,
    @NotBlank String audience,
    @NotNull Algorithm algorithm,
    @NotNull Duration clockSkew,
    @Valid @NotNull TokenTtl accessToken,
    @Valid @NotNull TokenTtl refreshToken,
    @Valid @NotNull Keys keys
) {

  public SecurityProps {
    // Cross-field sanity checks
    if (algorithm.isHmac()) {
      if (keys.hmacSecret() == null || keys.hmacSecret().isBlank()) {
        throw new IllegalArgumentException("security.jwt.keys.hmac-secret is required for HS* algorithms");
      }
      if (keys.publicKey() != null || keys.privateKey() != null) {
        throw new IllegalArgumentException("public/private keys must be empty for HS* algorithms");
      }
    } else { // RSA/EC
      if (keys.publicKey() == null || keys.privateKey() == null) {
        throw new IllegalArgumentException("public/private keys are required for RS*/ES* algorithms");
      }
      if (keys.hmacSecret() != null && !keys.hmacSecret().isBlank()) {
        throw new IllegalArgumentException("hmac-secret must be empty for RS*/ES* algorithms");
      }
    }
  }

  public enum Algorithm {
    HS256, RS256, ES256;
    public boolean isHmac() { return name().startsWith("HS"); }
  }

  public record TokenTtl(@NotNull Duration ttl) {}

  public record Keys(
      // For asymmetric modes:
      Resource publicKey,
      Resource privateKey,
      // For HMAC modes:
      String hmacSecret
  ) {}
}
```

---

## Enable binding (if not already scanning `config/`)

```java
// src/main/java/com/example/app/Application.java
package com.example.app;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;

@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { }
```

---

## Key material loader (bean for downstream services)

This sits in `config/` as glue: reads PEM/HMAC once, exposes typed keys.

```java
// src/main/java/com/example/app/config/props/JwtKeyProvider.java
package com.example.app.config.props;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.StreamUtils;

import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.security.*;
import java.security.interfaces.*;
import java.security.spec.*;
import java.util.Base64;

@Configuration
public class JwtKeyProvider {

  @Bean
  public JwtKeys jwtKeys(SecurityProps props) {
    try {
      return props.algorithm().isHmac()
          ? loadHmac(props)
          : loadAsymmetric(props);
    } catch (Exception e) {
      throw new IllegalStateException("Failed to load JWT keys", e);
    }
  }

  private JwtKeys loadHmac(SecurityProps p) {
    byte[] raw = p.keys().hmacSecret().getBytes(StandardCharsets.UTF_8);
    SecretKey key = new SecretKeySpec(raw, "HmacSHA256"); // HS256
    return JwtKeys.hmac(key);
  }

  private JwtKeys loadAsymmetric(SecurityProps p) throws Exception {
    String pubPem  = readAll(p.keys().publicKey());
    String prvPem  = readAll(p.keys().privateKey());

    PublicKey  pub = toPublicKey(pubPem, p.algorithm());
    PrivateKey prv = toPrivateKey(prvPem, p.algorithm());

    return JwtKeys.asymmetric(pub, prv);
  }

  private static String readAll(org.springframework.core.io.Resource r) throws Exception {
    try (InputStream in = r.getInputStream()) {
      return StreamUtils.copyToString(in, StandardCharsets.US_ASCII);
    }
  }

  private static PublicKey toPublicKey(String pem, SecurityProps.Algorithm alg) throws Exception {
    String base64 = pem.replaceAll("-----BEGIN (.*)-----", "")
                       .replaceAll("-----END (.*)-----", "")
                       .replaceAll("\\s", "");
    byte[] der = Base64.getDecoder().decode(base64);
    KeyFactory kf = KeyFactory.getInstance(alg.name().startsWith("RS") ? "RSA" : "EC");
    X509EncodedKeySpec spec = new X509EncodedKeySpec(der);
    return kf.generatePublic(spec);
  }

  private static PrivateKey toPrivateKey(String pem, SecurityProps.Algorithm alg) throws Exception {
    String base64 = pem.replaceAll("-----BEGIN (.*)-----", "")
                       .replaceAll("-----END (.*)-----", "")
                       .replaceAll("\\s", "");
    byte[] der = Base64.getDecoder().decode(base64);
    KeyFactory kf = KeyFactory.getInstance(alg.name().startsWith("RS") ? "RSA" : "EC");
    PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(der);
    return kf.generatePrivate(spec);
  }

  /** Simple carrier — place where your JWT lib expects it (Nimbus/JJWT/etc.). */
  public record JwtKeys(SecretKey hmac, PublicKey publicKey, PrivateKey privateKey) {
    public static JwtKeys hmac(SecretKey k) { return new JwtKeys(k, null, null); }
    public static JwtKeys asymmetric(PublicKey pub, PrivateKey prv) { return new JwtKeys(null, pub, prv); }
  }
}
```

> Swap the `SecretKeySpec` algorithm if you later support HS384/HS512.

---

## Example consumer (pseudo-service)

```java
// src/main/java/com/example/app/security/JwtService.java
package com.example.app.security;

im/propsport com.example.app.config.props.Jw/propstKeyProvider.JwtKeys;
import com.example.app.config.props.SecurityProps;
import org.springframework.stereotype.Service;

import java.time.Clock;
import java.time.Instant;

@Service
public class JwtService {
  private final SecurityProps props;
  private final JwtKeys keys;
  private final Clock clock;

  public JwtService(SecurityProps props, JwtKeys keys, Clock clock) {
    this.props = props;
    this.keys = keys;
    this.clock = clock;
  }

  public Instant now() { return clock.instant(); }

  // Sign/verify methods here using your chosen JWT lib, e.g. Nimbus or JJWT.
  // Use props.accessToken().ttl(), props.refreshToken().ttl(), props.clockSkew(), etc.
}
```

---

## ENV equivalents

```bash
SECURITY_JWT_ISSUER=com.example.app
SECURITY_JWT_AUDIENCE=shoppingcart-clients
SECURITY_JWT_ALGORITHM=RS256
SECURITY_JWT_CLOCK_SKEW=30s
SECURITY_JWT_ACCESS_TOKEN_TTL=15m
SECURITY_JWT_REFRESH_TOKEN_TTL=30d

# Asymmetric:
SECURITY_JWT_KEYS_PUBLIC=classpath:keys/jwt.pub.pem
SECURITY_JWT_KEYS_PRIVATE=file:/etc/keys/jwt.key.pem

# HMAC:
SECURITY_JWT_KEYS_HMAC_SECRET=${JWT_SECRET}   # inject via real secret store
```

---

## Guardrails

* **Choose one family**: HS* (single secret) **or** RS*/ES* (keypair).
  The constructor enforces mutual exclusivity.
* **Keep secrets out of Git**: leave `hmac-secret` empty in YAML; feed via env/secret manager.
* **Short access TTLs, long refresh TTLs**: e.g., `15m` vs `30d`.
* **Clock skew small**: `30s`–`2m`. Bigger skews suggest broken clocks.
* **PEM format**: keys must be standard PEM encodings (PKCS#8 private, X.509 public).

---

## Tests (binding + invariants)

```java
// src/test/java/com/example/app/config/props/SecurityPropsTest.java
package com.example.app.config.props;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SecurityPropsTest {

  private ApplicationContextRunner run(String... props) {
    return new ApplicationContextRunner()
        .withUserConfiguration(Bootstrap.class)
        .withPropertyValues(props);
  }

  @Test
  void requiresHmacSecretForHs() {
    assertThatThrownBy(() -> run(
        "security.jwt.issuer=ex",
        "security.jwt.audience=aud",
        "security.jwt.algorithm=HS256",
        "security.jwt.clock-skew=30s",
        "security.jwt.access-token.ttl=15m",
        "security.jwt.refresh-token.ttl=30d"
    ).run(c -> c.getBean(SecurityProps.class))).hasMessageContaining("hmac-secret is required");
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(SecurityProps.class)
  static class Bootstrap {}
}
```

---

## Quick checklist

* `@ConfigurationProperties(prefix="security.jwt")` bound and validated
* Algorithm family enforces **exclusive** key material
* TTLs and skew are `Duration` types, not strings
* Keys loaded once, reused via `JwtKeyProvider` bean
* Secrets come from ENV/secret store, not YAML

---

### Next pieces you may want

* `CorsConfig` defaults (tight dev/prod split)
* `PasswordPolicyProps` (min length, entropy, breach-check toggle)
* `SecurityFilterChain` skeleton wired to `SecurityProps`
