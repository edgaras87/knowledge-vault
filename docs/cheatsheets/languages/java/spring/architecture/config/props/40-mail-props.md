---
title: MailProps
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Centralize SMTP transport, TLS/auth, timeouts, pooling, sender identities, templates, and rate limits into a single validated record with rich types, enabling services to send email without string parsing.
aliases:
    - Spring Config Props Layer - MailProps Cheatsheet
---

# MailProps — typed SMTP + templating config

Centralize **transport**, **TLS/auth**, **timeouts**, **pooling**, **sender identities**, **templates**, and **rate limits**. Consumers get **typed** values—no string parsing.

---

## YAML we’re binding

```yaml
mail:
  enabled: true            # turn off to no-op mail sending in lower envs
  mock: false              # when true, log to files/console instead of real SMTP

  smtp:
    host: "smtp.example.com"
    port: 587
    username: "${SMTP_USER}"
    password: "${SMTP_PASS}"
    auth: true
    protocol: "smtp"       # smtp | smtps
    timeouts:
      connect: 5s
      read: 10s
      write: 10s

  tls:
    starttls: required     # disabled | opportunistic | required
    trust-all: false       # trust any cert (dev only)
    ssl: false             # wrap socket in SSL (legacy smtps)

  pool:
    enabled: true
    size: 10
    max-idle: 5
    eviction-ttl: 60s

  from:
    address: "noreply@example.com"
    name: "Shopping Cart"

  reply-to:
    address: "support@example.com"
    name: "Support"

  templates:
    base-path: "classpath:mail/templates/" # for text/html templates

  limits:
    max-attachments: 5
    max-total-attachment-size: 10MB
```

---

## Where it lives

* /props**Class:** `com.example.app.config.props.MailProps`
* **Used by:** `MailConfig` (builds `JavaMailSender`), `EmailService`, templating engines

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/props/MailProps.java
package com.example.app.config.props;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.util.unit.DataSize;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;

@Validated
@ConfigurationProperties(prefix = "mail")
public record MailProps(
    boolean enabled,
    boolean mock,
    @Valid @NotNull Smtp smtp,
    @Valid @NotNull Tls tls,
    @Valid @NotNull Pool pool,
    @Valid @NotNull Identity from,
    @Valid Identity replyTo,
    @Valid @NotNull Templates templates,
    @Valid @NotNull Limits limits
) {
  public MailProps {
    if (smtp.port() < 1 || smtp.port() > 65535) {
      throw new IllegalArgumentException("mail.smtp.port must be 1..65535");
    }
    if (limits.maxAttachments() < 0) {
      throw new IllegalArgumentException("mail.limits.max-attachments must be >= 0");
    }
    if (tls.starttls() == Tls.StartTls.required && tls.ssl()) {
      throw new IllegalArgumentException("startTLS=required and ssl=true are mutually exclusive");
    }
    if (smtp.auth() && (smtp.username() == null || smtp.username().isBlank())) {
      throw new IllegalArgumentException("SMTP auth=true requires username/password");
    }
  }

  // --- nested ---

  public record Smtp(
      @NotBlank String host,
      @Min(1) @Max(65535) int port,
      String username,
      String password,
      boolean auth,
      @NotBlank String protocol,   // "smtp" or "smtps"
      @Valid @NotNull Timeouts timeouts
  ) {
    public record Timeouts(
        @NotNull Duration connect,
        @NotNull Duration read,
        @NotNull Duration write
    ) {}
  }

  public record Tls(
      @NotNull StartTls starttls,
      boolean trustAll,
      boolean ssl
  ) {
    public enum StartTls { disabled, opportunistic, required }
  }

  public record Pool(
      boolean enabled,
      @Min(1) int size,
      @Min(0) int maxIdle,
      @NotNull Duration evictionTtl
  ) {}

  public record Identity(
      @Email @NotBlank String address,
      @NotBlank String name
  ) {}

  public record Templates(
      @NotBlank String basePath
  ) {}

  public record Limits(
      @Min(0) int maxAttachments,
      @NotNull DataSize maxTotalAttachmentSize
  ) {}
}
```

---

## Wiring a `JavaMailSender` (Jakarta Mail) from props

```java
// src/main/java/com/example/app/config/props/MailConfig.java
package com.example.app.config.props;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.JavaMailSenderImpl;

import java.util.Properties;

@Configuration
public class MailConfig {

  @Bean
  public JavaMailSender mailSender(MailProps p) {
    var impl = new JavaMailSenderImpl();

    impl.setHost(p.smtp().host());
    impl.setPort(p.smtp().port());
    impl.setProtocol(p.smtp().protocol());
    if (p.smtp().username() != null) impl.setUsername(p.smtp().username());
    if (p.smtp().password() != null) impl.setPassword(p.smtp().password());

    Properties props = impl.getJavaMailProperties();
    props.put("mail.transport.protocol", p.smtp().protocol());
    props.put("mail.smtp.auth", String.valueOf(p.smtp().auth()));
    props.put("mail.smtp.connectiontimeout", String.valueOf(p.smtp().timeouts().connect().toMillis()));
    props.put("mail.smtp.timeout", String.valueOf(p.smtp().timeouts().read().toMillis()));
    props.put("mail.smtp.writetimeout", String.valueOf(p.smtp().timeouts().write().toMillis()));

    // TLS / SSL
    switch (p.tls().starttls()) {
      case disabled -> props.put("mail.smtp.starttls.enable", "false");
      case opportunistic, required -> {
        props.put("mail.smtp.starttls.enable", "true");
        props.put("mail.smtp.starttls.required", String.valueOf(p.tls().starttls() == MailProps.Tls.StartTls.required));
      }
    }
    if (p.tls().ssl()) {
      props.put("mail.smtp.ssl.enable", "true"); // legacy SMTPS
    }
    if (p.tls().trustAll()) {
      // Dev only: trust all certificates
      props.put("mail.smtp.ssl.trust", "*");
    }

    // Connection pool (Jakarta Mail native pool is limited; still set hints)
    if (p.pool().enabled()) {
      props.put("mail.smtp.pool", "true");
      props.put("mail.smtp.pool.size", String.valueOf(p.pool().size()));
    }

    return impl;
  }
}
```

---

## Usage sketch (service)

```java
// src/main/java/com/example/app/mail/EmailService.java
package com.example.app.mail;

im/propsport com.example.app.config.props.MailProps;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;

@Service
public class EmailService {
  private final JavaMailSender sender;
  private final MailProps props;

  public EmailService(JavaMailSender sender, MailProps props) {
    this.sender = sender;
    this.props = props;
  }

  public void sendPlain(String to, String subject, String body) {
    if (!props.enabled()) return;  // no-op when disabled

    var msg = new SimpleMailMessage();
    msg.setFrom(props.from().address());
    msg.setTo(to);
    msg.setSubject(subject);
    msg.setText(body);
    if (props.replyTo() != null) {
      msg.setReplyTo(props.replyTo().address());
    }

    if (props.mock()) {
      // write to log/file instead of SMTP
      System.out.printf("[MOCK MAIL] to=%s subj=%s\n%s\n", to, subject, body);
      return;
    }

    sender.send(msg);
  }
}
```

---

## ENV equivalents

```bash
MAIL_ENABLED=true
MAIL_MOCK=false

MAIL_SMTP_HOST=smtp.example.com
MAIL_SMTP_PORT=587
MAIL_SMTP_USERNAME=${SMTP_USER}
MAIL_SMTP_PASSWORD=${SMTP_PASS}
MAIL_SMTP_AUTH=true
MAIL_SMTP_PROTOCOL=smtp
MAIL_SMTP_TIMEOUTS_CONNECT=5s
MAIL_SMTP_TIMEOUTS_READ=10s
MAIL_SMTP_TIMEOUTS_WRITE=10s

MAIL_TLS_STARTTLS=required
MAIL_TLS_TRUST_ALL=false
MAIL_TLS_SSL=false

MAIL_POOL_ENABLED=true
MAIL_POOL_SIZE=10
MAIL_POOL_MAX_IDLE=5
MAIL_POOL_EVICTION_TTL=60s

MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="Shopping Cart"
MAIL_REPLY_TO_ADDRESS=support@example.com
MAIL_REPLY_TO_NAME=Support

MAIL_TEMPLATES_BASE_PATH=classpath:mail/templates/

MAIL_LIMITS_MAX_ATTACHMENTS=5
MAIL_LIMITS_MAX_TOTAL_ATTACHMENT_SIZE=10MB
```

---

## Guardrails & notes

* **Choose one transport**: modern STARTTLS on port **587** (`starttls=required`) is typical; legacy SSL (`ssl=true`) is rare now.
* **Auth on?** Supply `username/password` via **ENV / secret store**, not YAML.
* **Mock mode** is gold for dev/test; pair with integration tests that assert on “outbox” files/logs.
* **Attachment limits** should align with **web upload** limits and **StorageProps**.
* If you template HTML, set a single **template base path** and keep both `*.html` and `*.txt` side by side.

---

## Tests (binding + invariants)

```java
// src/test/java/com/example/app/config/props/MailPropsTest.java
package com.example.app.config.props;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class MailPropsTest {

  private ApplicationContextRunner run(String... p) {
    return new ApplicationContextRunner()
        .withUserConfiguration(Bootstrap.class)
        .withPropertyValues(p);
  }

  @Test
  void starttlsAndSslAreMutuallyExclusive() {
    assertThatThrownBy(() -> run(
        "mail.enabled=true",
        "mail.mock=false",
        "mail.smtp.host=smtp.example.com",
        "mail.smtp.port=587",
        "mail.smtp.auth=true",
        "mail.smtp.username=user",
        "mail.smtp.password=pass",
        "mail.smtp.protocol=smtp",
        "mail.smtp.timeouts.connect=5s",
        "mail.smtp.timeouts.read=5s",
        "mail.smtp.timeouts.write=5s",
        "mail.tls.starttls=required",
        "mail.tls.ssl=true",
        "mail.tls.trust-all=false",
        "mail.pool.enabled=true",
        "mail.pool.size=5",
        "mail.pool.max-idle=2",
        "mail.pool.eviction-ttl=60s",
        "mail.from.address=noreply@example.com",
        "mail.from.name=App",
        "mail.templates.base-path=classpath:mail/templates/",
        "mail.limits.max-attachments=3",
        "mail.limits.max-total-attachment-size=10MB"
    ).run(c -> c.getBean(MailProps.class))).hasMessageContaining("mutually exclusive");
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(MailProps.class)
  static class Bootstrap {}
}
```
