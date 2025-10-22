---
title: Spring Annotations — Future Blueprint
date: 2025-10-22
summary: A blueprint for organizing cheatsheets on Spring annotations by capability, ensuring scalability and coherence across the evolving Spring ecosystem.
---



# 🧩 Blueprint — Spring Annotations Cheatsheets

**Goal:** Create a modular, future-proof structure for documenting **Spring annotations**, grouped by capability (not by module).  
This ensures scalability as the ecosystem grows (Boot, WebFlux, Security, Kafka, etc.) without fragmenting related topics.

---

## 📁 Folder Layout

```

cheatsheets/
└─ frameworks/
└─ spring/
└─ annotations/
├─ index.md                       # master overview & quick links
├─ core-di/
│  ├─ stereotypes.md              # @Component, @Service, @Repository, @Controller
│  ├─ configuration-beans.md      # @Configuration, @Bean, @Import, @Lazy, @Scope
│  ├─ wiring-and-profiles.md      # @Autowired, @Qualifier, @Primary, @Value, @Profile
│  ├─ conditions.md               # @Conditional*, meta-annotations
│  └─ lifecycle-events.md         # @PostConstruct, @PreDestroy, @EventListener
├─ mvc-rest/
│  ├─ controller-basics.md        # @Controller, @RestController, @ControllerAdvice
│  ├─ mapping.md                  # @RequestMapping, @Get/Post/Put/Delete/PatchMapping
│  ├─ params-and-bodies.md        # @PathVariable, @RequestParam, @RequestBody, @ResponseBody
│  ├─ responses-and-errors.md     # @ResponseStatus, @ExceptionHandler, @ResponseEntity*
│  └─ cors-and-versioning.md      # @CrossOrigin, versioning patterns
├─ webflux/
│  ├─ enable-and-config.md        # @EnableWebFlux, functional routing notes
│  └─ controller-notes.md         # shared controller annotations + reactive caveats
├─ validation/
│  ├─ method-level.md             # @Validated, @Valid
│  └─ bean-constraints.md         # Jakarta validation annotations
├─ data/
│  ├─ repositories.md             # @Repository, @EnableJpaRepositories, @EnableMongoRepositories
│  ├─ jpa-entity-auditing.md      # @Entity*, @Id*, @CreatedDate, @Version, @EnableJpaAuditing
│  ├─ queries.md                  # @Query, @Modifying, projections
│  └─ converters.md               # @ReadingConverter, @WritingConverter
├─ transactions/
│  ├─ transactional-basics.md     # @Transactional, propagation/isolation
│  └─ enable-tx.md                # @EnableTransactionManagement
├─ security/
│  ├─ enable-and-config.md        # @EnableWebSecurity, @EnableMethodSecurity
│  ├─ method-security.md          # @PreAuthorize, @PostAuthorize, @Secured, @RolesAllowed
│  └─ auth-context.md             # @AuthenticationPrincipal, @CurrentSecurityContext
├─ async-scheduling/
│  ├─ async.md                    # @Async, thread pool notes
│  └─ scheduling.md               # @Scheduled, @EnableScheduling, cron
├─ caching/
│  └─ cache-annotations.md        # @EnableCaching, @Cacheable, @CachePut, @CacheEvict, @Caching
├─ messaging/
│  ├─ kafka.md                    # @EnableKafka, @KafkaListener, @SendTo
│  ├─ rabbitmq.md                 # @EnableRabbit, @RabbitListener
│  └─ jms.md                      # @JmsListener
├─ actuator/
│  └─ endpoints.md                # @Endpoint, @ReadOperation, @WriteOperation
└─ boot/
├─ application-and-auto.md     # @SpringBootApplication, @AutoConfiguration
├─ config-properties.md        # @ConfigurationProperties, @EnableConfigurationProperties
└─ conditional-on.md           # @ConditionalOnProperty/Bean/Class/MissingBean/Resource

```

> `ResponseEntity` is not an annotation but belongs conceptually to REST responses — keep a mini reference for context.

---

## 🧠 Philosophy

1. **Organize by capability**, not dependency.  
   Jump straight to “transactions” or “security” without recalling which starter or module they live in.

2. **Keep pages thin.** Each `.md` covers one concept cluster (e.g., *mappings*, *transactions*).  
   Avoid giant scrolls; prefer clear entry points.

3. **Cross-link ruthlessly.** Example:  
   - `@Transactional` → links to `data/queries.md`  
   - `@PreAuthorize` → links to `security/method-security.md` and `core-di/conditions.md`

4. **Reactive-ready.** WebFlux lives in its own corner but reuses MVC annotations; highlight differences, don’t duplicate content.

5. **Boot awareness without dependency.** Boot annotations live under `/boot`, but pages cross-reference corresponding Core features.

---

## 🪶 Page Template (per-annotation file)

```markdown
# @AnnotationName — Cheatsheet

**Essence:** One-line summary of its purpose.  
**When to use:** Typical scenario or trigger condition.  
**Gotchas:** Common pitfalls or side effects.

## 1) Minimal Example

```java
// 10–20 lines max, compile-ready
```

**Effect:** Short description of runtime behavior.

## 2) Options that Matter

| Option | Default | What It Changes | Use When      |
| ------ | ------- | --------------- | ------------- |
| name   | —       | Bean name       | Custom wiring |

## 3) Interactions

* Works with: @A, @B
* Conflicts with: @C (explain why)
* Boot tie-ins: @ConditionalOnProperty("...")

## 4) Troubleshooting

* Symptom → Likely cause → Fix

**See also:** Links to related pages.



Optional front-matter for search and freshness tracking:

```yaml
---
title: "@Transactional"
category: "transactions"
tags: ["propagation","isolation","rollback"]
tested_on: ["Spring Boot 3.3","Spring 6"]
last_update: 2025-10-22
---
```

---

## 🗂 Canonical Annotation Sets

### Core & DI

`@Component`, `@Service`, `@Repository`, `@Controller`,
`@Configuration`, `@Bean`, `@Import`, `@Scope`, `@Lazy`,
`@Autowired`, `@Qualifier`, `@Primary`, `@Value`, `@Profile`,
`@Conditional`, `@PostConstruct`, `@PreDestroy`, `@EventListener`

### MVC / REST

`@Controller`, `@RestController`, `@ControllerAdvice`,
`@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`,
`@PathVariable`, `@RequestParam`, `@RequestBody`, `@ResponseBody`,
`@ResponseStatus`, `@ExceptionHandler`, `@CrossOrigin`

### WebFlux

`@EnableWebFlux`, reactive controller differences (same annotations)

### Validation

`@Validated`, `@Valid`, plus Jakarta annotations (`@NotNull`, `@Size`, `@Email`…)

### Data

`@Repository`, `@EnableJpaRepositories`, `@EnableMongoRepositories`,
`@Query`, `@Modifying`,
`@CreatedDate`, `@LastModifiedDate`, `@EnableJpaAuditing`,
`@Entity`, `@Id`, `@GeneratedValue`, `@Version`

### Transactions

`@Transactional`, `@EnableTransactionManagement`

### Security

`@EnableWebSecurity`, `@EnableMethodSecurity`,
`@PreAuthorize`, `@PostAuthorize`, `@Secured`, `@RolesAllowed`,
`@AuthenticationPrincipal`, `@CurrentSecurityContext`

### Async / Scheduling

`@Async`, `@EnableAsync`,
`@Scheduled`, `@EnableScheduling`

### Caching

`@EnableCaching`, `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`

### Messaging

Kafka → `@EnableKafka`, `@KafkaListener`, `@SendTo`
RabbitMQ → `@EnableRabbit`, `@RabbitListener`
JMS → `@JmsListener`

### Actuator

`@Endpoint`, `@ReadOperation`, `@WriteOperation`

### Boot

`@SpringBootApplication`, `@AutoConfiguration`,
`@ConfigurationProperties`, `@EnableConfigurationProperties`,
`@ConditionalOnProperty`, `@ConditionalOnBean`, `@ConditionalOnClass`, etc.

---

## 🧭 `index.md` Template (master overview)


# Spring Annotations — Master Map

Jump by capability:

- Core & DI → [Stereotypes](core-di/stereotypes.md) · [Beans](core-di/configuration-beans.md) · [Profiles](core-di/wiring-and-profiles.md)
- MVC / REST → [Controllers](mvc-rest/controller-basics.md) · [Mappings](mvc-rest/mapping.md) · [Errors](mvc-rest/responses-and-errors.md)
- WebFlux → [Enable & Config](webflux/enable-and-config.md)
- Validation → [@Valid & @Validated](validation/method-level.md)
- Data → [Repositories](data/repositories.md) · [Queries](data/queries.md)
- Transactions → [@Transactional](transactions/transactional-basics.md)
- Security → [Method Security](security/method-security.md)
- Async / Scheduling → [@Async](async-scheduling/async.md) · [@Scheduled](async-scheduling/scheduling.md)
- Caching → [Cache annotations](caching/cache-annotations.md)
- Messaging → [Kafka](messaging/kafka.md) · [RabbitMQ](messaging/rabbitmq.md)
- Actuator → [Custom endpoints](actuator/endpoints.md)
- Boot → [Auto-Config](boot/application-and-auto.md) · [ConfigurationProperties](boot/config-properties.md)

> **Legend:** Each page starts with “Essence / When to use / Gotchas” and ends with “Troubleshooting / See also”.


---

## 🧩 Maintenance Discipline

* **One page = one concept cluster.**
  Avoid “everything about @Controller in 500 lines” pages.

* **Short code, clear output.**
  Examples must compile and demonstrate only the annotation’s core purpose.

* **Cross-references are mandatory.**
  Use links liberally to show relationships and dependencies.

* **Track freshness.**
  Use `last_update:` and framework version tags for periodic review when Spring 7+ arrives.

---

✅ **Outcome:**
A clean, extensible system for documenting Spring annotations — capable of absorbing future frameworks (Micronaut, Quarkus) by mirroring this blueprint under their own namespaces.

