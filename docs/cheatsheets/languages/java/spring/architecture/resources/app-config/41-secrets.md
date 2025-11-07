---
title: Secrets 
date: 2025-11-06
tags: 
    - spring
    - spring-boot
    - configuration
    - properties
    - cheatsheet
summary: A cheat sheet for handling secrets in Spring Boot applications, detailing best practices for loading, storing, and managing sensitive configuration data securely.
aliases:
  - Spring Properties - Secrets Cheatsheet
---

# Secrets — Cheatsheet

Secrets include passwords, API keys, tokens, private hosts, encryption keys, and anything not meant for public or logged output.  
Spring can load secrets from environment variables, external files, or secret managers.  
This page covers the safe, minimal patterns for handling secrets without overengineering.

---

# 1. What counts as a secret?

- Database passwords  
- API keys (Stripe, AWS, Mailgun, etc.)  
- Private tokens (JWT signing key, OAuth client secret)  
- SSH keys or certificates  
- Internal service credentials  
- Admin credentials  

Not secrets:
- Public hosts  
- Feature flags  
- Pagination limits  
- File paths  
- Toggles  

---

# 2. Rule: **Secrets never belong in application.yml**

✅ Good:

```bash
export MAILGUN_KEY="live_xxxxxx"
```

❌ Bad:

```yaml
mailgun:
  key: live_xxxxxx   # do NOT commit this
```

YAML is version-controlled → confidential data must stay outside of it.

---

# 3. How to load secrets safely (environment variables)

Environment variables are the simplest, safest cross-platform approach.

Example:

```bash
export DB_PASSWORD="secure123!"
export JWT_SECRET="0123456789abcdef"
```

YAML references them:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost/app
    username: app
    password: ${DB_PASSWORD}
jwt:
  secret: ${JWT_SECRET}
```

If they’re missing, Spring fails at startup.

---

# 4. Docker / Compose — environment section

```yaml
services:
  api:
    image: myapp
    environment:
      DB_PASSWORD: ${DB_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
```

Compose overrides everything in application.yml.  
This is why secrets must not appear in YAML → Compose would leak them.

---

# 5. Where to put secret-holding config classes

Use a dedicated section:

```java
@ConfigurationProperties("secrets")
public record SecretsProps(String jwtSecret, String dbPassword) {}
```

Or organise per domain:

```java
@ConfigurationProperties("jwt")
public record JwtProps(String secret) {}
```

Remember:
- Never expose these via API DTOs  
- Never log them  
- Never put them inside `AppInfoResponse`  

Location:

```
src/main/java/.../config/
```

---

# 6. Testing secrets

Local tests can override secrets:

```java
@SpringBootTest
@TestPropertySource(properties = {
  "jwt.secret=test-secret"
})
class JwtTest {}
```

Never use real secrets in tests.

Alternatively, rely on environment variables:

```bash
JWT_SECRET=test-secret ./mvnw test
```

---

# 7. Default values (safe patterns)

Default values should *NOT* contain a real secret.

✅ Acceptable for dev-only defaults:

```yaml
jwt:
  secret: "dev-secret"
```

But only if:

- file is not used in production  
- value is not powerful  
- value is clearly dev-only  

For production: always use env vars.

---

# 8. Using external files (optional)

Spring allows referencing `.txt` files:

```yaml
jwt:
  secret: ${file:/run/secrets/jwt-secret}
```

Used commonly with:

- Docker Swarm secrets  
- Kubernetes secrets  
- Vault sidecar templates  

Useful in environments where env vars are discouraged.

---

# 9. Secret managers (future, optional)

Not needed for small deployments, but available:

- HashiCorp Vault  
- AWS Secrets Manager  
- GCP Secret Manager  
- Azure KeyVault  

These integrate with Spring via starters, injecting secrets directly into the Environment.

Only consider these when:

- multiple microservices consume secrets  
- secrets rotate automatically  
- you have ops tooling  

For now, environment variables are simpler and safer.

---

# 10. Logging: never log secrets

Avoid:

```java
log.info("Using DB password: {}", props.password());
```

Masking approach:

```java
log.debug("Using DB user: {}", user);
log.debug("Using DB password: <masked>");
```

Actuator’s `/env` endpoint also masks common secret names automatically.

---

# 11. Exposing secrets — never do it

Your API must *never* return secrets:

```json
{
  "jwtSecret": "xxx"
}
```

Even internal endpoints should avoid returning secrets.  
If a client needs to know a public value (e.g., public JWKS key), expose only that.

---

# 12. Practical example (aligned with your structure)

YAML:

```yaml
jwt:
  secret: ${JWT_SECRET}
```

Java:

```java
@ConfigurationProperties("jwt")
public record JwtProps(String secret) {}
```

Controller (incorrect):

```java
return new JwtInfoResponse(props.secret());  // ❌ never expose
```

Correct usage:

```java
String token = jwtService.issueToken(userId, props.secret());
```

No outbound exposure.

---

# 13. DO / DON'T / pitfalls

**DO**
- load secrets through environment variables  
- separate secrets into their own Properties classes  
- mask logs  
- use local dummy secrets for dev environment  
- fail fast if a secret is missing  

**DON'T**
- don’t commit secrets to Git  
- don’t store secrets in `application.yml`  
- don’t expose secrets via DTOs  
- don’t echo secrets in logging or exception messages  
- don’t hardcode secrets into code  

**Pitfalls**
- Using Compose environment → accidentally overriding intended values  
- Missing secrets causing Spring to substitute empty strings  
- Trailing spaces in environment variable values  
- Accidentally exposing secrets via `/actuator/env` (disable in production)

---

# 14. Quick glossary

- **Secret** — any value that must not be public.  
- **Env var** — standard OS-level secret source.  
- **Externalized config** — config outside the JAR.  
- **Secret manager** — service storing sealed credentials.  
- **Masking** — removing sensitive values from logs and responses.
