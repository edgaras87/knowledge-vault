---
title: StorageProps
date: 2025-11-09
tags: 
    - spring
    - spring-boot
    - configuration
    - architecture
    - cheatsheet
summary: Centralize object storage settings into a single validated record with rich types, enabling services to handle local or S3-compatible storage with multipart uploads, signed URLs, quotas, and content-type policies without string parsing.
aliases:
    - Spring Config Props Layer - StorageProps Cheatsheet
---



# StorageProps — typed object storage config (local | S3-compatible)

Centralize **backend choice**, **bucket/base-dir**, **multipart limits**, **signed URLs**, **quotas**, and **content-type policy**. Services consume **typed** values—no ad-hoc parsing.

---

## YAML we’re binding

```yaml
storage:
  backend: s3              # local | s3

  # Local backend
  base-dir: "/var/app/uploads"   # required when backend=local

  # S3 / S3-compatible (MinIO, Cloudflare R2, etc.)
  s3:
    endpoint: "https://s3.eu-central-1.amazonaws.com"
    region: "eu-central-1"
    bucket: "my-bucket"
    path-style-access: true       # MinIO often needs true
    credentials:
      access-key: "${S3_ACCESS_KEY}"
      secret-key: "${S3_SECRET_KEY}"

  multipart:
    enabled: true
    threshold: 8MB                # switch to MPU above this size
    part-size: 16MB               # S3 min 5MB; tune for throughput
    max-parts: 10000              # S3 max

  limits:
    max-object-size: 512MB
    total-space-quota: 100GB      # optional logical quota

  signed-urls:
    enabled: true
    ttl: 15m

  temp:
    dir: "${java.io.tmpdir}"
    clean-ttl: 1h

  allowed-content-types:
    - "image/*"
    - "application/pdf"

  antivirus:
    enabled: false
    host: "localhost"
    port: 3310
    timeout: 10s
```

---

## Where it lives

* **Class:** `com.example.app.config.props.StorageProps`
* **Used by:** `StorageService` (local/S3 implementations), upload controllers, background cleaners, virus-scan hooks

---

## Implementation (production-ready)

```java
// src/main/java/com/example/app/config/props/StorageProps.java
package com.example.app.config.props;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.core.io.Resource;
import org.springframework.validation.annotation.Validated;
import org.springframework.util.unit.DataSize;

import java.net.URI;
import java.nio.file.Path;
import java.time.Duration;
import java.util.List;

@Validated
@ConfigurationProperties(prefix = "storage")
public record StorageProps(
    @NotNull Backend backend,

    // local
    Path baseDir,

    // s3
    @Valid S3 s3,

    // cross-cutting policies
    @Valid @NotNull Multipart multipart,
    @Valid @NotNull Limits limits,
    @Valid @NotNull SignedUrls signedUrls,
    @Valid @NotNull Temp temp,

    @NotNull List<@NotBlank String> allowedContentTypes,

    @Valid @NotNull Antivirus antivirus
) {
  public StorageProps {
    // Backend-specific requirements
    if (backend == Backend.local) {
      if (baseDir == null) {
        throw new IllegalArgumentException("storage.base-dir is required when backend=local");
      }
    } else if (backend == Backend.s3) {
      if (s3 == null) throw new IllegalArgumentException("storage.s3 block is required when backend=s3");
      if (s3.bucket == null || s3.bucket.isBlank())
        throw new IllegalArgumentException("storage.s3.bucket is required");
      if (s3.endpoint == null)
        throw new IllegalArgumentException("storage.s3.endpoint is required");
      if (s3.region == null || s3.region.isBlank())
        throw new IllegalArgumentException("storage.s3.region is required");
      if (s3.credentials == null
          || s3.credentials.accessKey == null || s3.credentials.accessKey.isBlank()
          || s3.credentials.secretKey == null || s3.credentials.secretKey.isBlank()) {
        throw new IllegalArgumentException("storage.s3.credentials.access-key/secret-key are required");
      }
      // S3 MPU constraints
      if (multipart.enabled) {
        var min = DataSize.ofMegabytes(5);      // S3 minimum part size
        var max = DataSize.ofGigabytes(5);      // S3 maximum part size
        if (multipart.partSize.toBytes() < min.toBytes() || multipart.partSize.toBytes() > max.toBytes()) {
          throw new IllegalArgumentException("storage.multipart.part-size must be between 5MB and 5GB for S3");
        }
        if (multipart.maxParts < 1 || multipart.maxParts > 10_000) {
          throw new IllegalArgumentException("storage.multipart.max-parts must be 1..10000");
        }
      }
    }

    // Generic constraints
    if (limits.maxObjectSize.toBytes() <= 0) {
      throw new IllegalArgumentException("storage.limits.max-object-size must be > 0");
    }
    if (allowedContentTypes.isEmpty()) {
      throw new IllegalArgumentException("storage.allowed-content-types must contain at least one entry");
    }
    if (signedUrls.enabled && (signedUrls.ttl == null || signedUrls.ttl.isZero() || signedUrls.ttl.isNegative())) {
      throw new IllegalArgumentException("storage.signed-urls.ttl must be positive when enabled");
    }
  }

  public enum Backend { local, s3 }

  // --- nested records ---

  public record S3(
      @NotNull URI endpoint,
      @NotBlank String region,
      @NotBlank String bucket,
      boolean pathStyleAccess,
      @Valid @NotNull Credentials credentials
  ) {
    public record Credentials(
        @NotBlank String accessKey,
        @NotBlank String secretKey // keep out of git; prefer env/secret store
    ) {}
  }

  public record Multipart(
      boolean enabled,
      @NotNull DataSize threshold,
      @NotNull DataSize partSize,
      @Min(1) @Max(10_000) int maxParts
  ) {}

  public record Limits(
      @NotNull DataSize maxObjectSize,
      DataSize totalSpaceQuota // optional; null = unlimited
  ) {}

  public record SignedUrls(
      boolean enabled,
      Duration ttl
  ) {}

  public record Temp(
      Path dir,
      @NotNull Duration cleanTtl
  ) {}

  public record Antivirus(
      boolean enabled,
      @NotBlank String host,
      @Min(1) @Max(65535) int port,
      @NotNull Duration timeout
  ) {}
}
```

---

## Enable binding (if you aren’t already scanning `config/`)

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

## Usage sketches

### 1) S3 client wiring (AWS SDK v2 / MinIO)

```java
// src/main/java/com/example/app/config/props/S3ClientConfig.java
package com.example.app.config.props;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.S3Configuration;
import software.amazon.awssdk.http.urlconnection.UrlConnectionHttpClient;

import java.net.URI;

@Configuration
public class S3ClientConfig {

  @Bean
  public S3Client s3Client(StorageProps props) {
    if (props.backend() != StorageProps.Backend.s3) return null;

    var s3 = props.s3();
    var creds = StaticCredentialsProvider.create(
        AwsBasicCredentials.create(s3.credentials().accessKey(), s3.credentials().secretKey())
    );

    var cfg = S3Configuration.builder()
        .pathStyleAccessEnabled(s3.pathStyleAccess())
        .build();

    return S3Client.builder()
        .credentialsProvider(creds)
        .region(Region.of(s3.region()))
        .endpointOverride(URI.create(s3.endpoint().toString()))
        .serviceConfiguration(cfg)
        .httpClientBuilder(UrlConnectionHttpClient.builder())
        .build();
  }
}
```

### 2) Local storage service vs S3 storage service

Define a small port:

```java
// src/main/java/com/example/app/storage/StorageService.java
package com.example.app.storage;

import java.io.InputStream;
import java.net.URI;

public interface StorageService {
  String put(String key, InputStream data, long length, String contentType);
  InputStream get(String key);
  void delete(String key);
  URI presignGet(String key); // may throw UnsupportedOperationException if disabled
}
```

Then create two beans (not shown in full):

* `LocalStorageService` using `StorageProps.baseDir()` and `Files.copy`
* `S3StorageService` using `S3Client`, `putObject`, `presigner` respecting `signedUrls.ttl()`, `multipart.*`, and `limits.maxObjectSize()`

Pick with a `@ConditionalOnProperty` or by injecting `StorageProps` and branching in a `@Configuration`.

---

## ENV equivalents

```bash
STORAGE_BACKEND=s3

# Local
STORAGE_BASE_DIR=/var/app/uploads

# S3 block
STORAGE_S3_ENDPOINT=https://s3.eu-central-1.amazonaws.com
STORAGE_S3_REGION=eu-central-1
STORAGE_S3_BUCKET=my-bucket
STORAGE_S3_PATH_STYLE_ACCESS=true
STORAGE_S3_CREDENTIALS_ACCESS_KEY=${S3_ACCESS_KEY}
STORAGE_S3_CREDENTIALS_SECRET_KEY=${S3_SECRET_KEY}

# Multipart
STORAGE_MULTIPART_ENABLED=true
STORAGE_MULTIPART_THRESHOLD=8MB
STORAGE_MULTIPART_PART_SIZE=16MB
STORAGE_MULTIPART_MAX_PARTS=10000

# Limits
STORAGE_LIMITS_MAX_OBJECT_SIZE=512MB
STORAGE_LIMITS_TOTAL_SPACE_QUOTA=100GB

# Signed URLs
STORAGE_SIGNED_URLS_ENABLED=true
STORAGE_SIGNED_URLS_TTL=15m

# Temp
STORAGE_TEMP_DIR=${java.io.tmpdir}
STORAGE_TEMP_CLEAN_TTL=1h

# Content types
STORAGE_ALLOWED_CONTENT_TYPES[0]=image/*
STORAGE_ALLOWED_CONTENT_TYPES[1]=application/pdf

# Antivirus
STORAGE_ANTIVIRUS_ENABLED=false
STORAGE_ANTIVIRUS_HOST=localhost
STORAGE_ANTIVIRUS_PORT=3310
STORAGE_ANTIVIRUS_TIMEOUT=10s
```

---

## Guardrails & practical tips

* **One backend at a time.** Constructor enforces required fields per backend.
* **S3 MPU constraints**: `part-size` **≥ 5 MB** and **≤ 5 GB**; **≤ 10 000 parts**.
* **Max object size** should be ≥ `multipart.threshold` and large enough for your biggest uploads.
* **`allowed-content-types`** supports wildcards like `image/*`; enforce strictly at the web edge.
* **Secrets never in Git**: feed S3 credentials via ENV/secret manager.
* **MinIO** usually needs `path-style-access=true`.
* **Signed URLs**: if disabled, your `StorageService.presignGet` should throw `UnsupportedOperationException`.

---

## Tests (binding + invariants)

```java
// src/test/java/com/example/app/config/props/StoragePropsTest.java
package com.example.app.config.props;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class StoragePropsTest {

  private ApplicationContextRunner run(String... props) {
    return new ApplicationContextRunner()
      .withUserConfiguration(Bootstrap.class)
      .withPropertyValues(props);
  }

  @Test
  void requiresBaseDirForLocal() {
    assertThatThrownBy(() -> run(
      "storage.backend=local",
      "storage.multipart.enabled=false",
      "storage.limits.max-object-size=10MB",
      "storage.signed-urls.enabled=false",
      "storage.temp.clean-ttl=1h",
      "storage.allowed-content-types[0]=application/octet-stream",
      "storage.antivirus.enabled=false",
      "storage.antivirus.host=localhost",
      "storage.antivirus.port=3310",
      "storage.antivirus.timeout=5s"
    ).run(c -> c.getBean(StorageProps.class))).hasMessageContaining("base-dir is required");
  }

  @org.springframework.context.annotation.Configuration
  @org.springframework.boot.context.properties.EnableConfigurationProperties(StorageProps.class)
  static class Bootstrap {}
}
```

