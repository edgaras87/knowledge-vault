---
title: i18n Future Blueprint
date: 2025-11-10 
summary: A future-proof blueprint for organizing internationalization (i18n) in a Spring application, scaling from solo development to team environments with multiple locales and CI checks.
---

# i18n — Future Blueprint (v1 → v3)

Here’s a **future-proof blueprint** for your `i18n/` stack so it scales cleanly from “solo dev” to “team + multiple locales + CI gates”. It’s opinionated, minimal, and matches your current setup (`Messages`, `ProblemKey`, `MessageCodes`, `@Message` injector, `/problems` docs).

## Goals

* Stable, type-safe keys with ergonomic access.
* One source of truth for problems (`ProblemKey`) that also drives docs.
* CI checks that prevent broken/missing translations.
* Simple path to add locales and manage deprecations.

---

## Phase v1 — Solid foundation (you’re ~here)

### Folder map

```
src/main/java/com/example/app/i18n/
├─ Messages.java
├─ MessageCodes.java          # or standalone ProblemKey.java
├─ ProblemKey.java            # (preferred standalone)
├─ Message.java               # @Message qualifier
└─ LocalizedMessage.java
src/main/java/com/example/app/config/
├─ MessageSourceConfig.java
└─ MessageInjectionPostProcessor.java
src/main/resources/i18n/
├─ messages.properties
├─ problem-messages.properties
└─ validation.properties
```

### Conventions

* Keys: `app.*`, `problem.<slug>.(title|detail)`, `validation.*`.
* Slugs: `kebab-case`, stable, become `/problems/{slug}`.
* Locale source: `Accept-Language` via `WebConfig` resolver.
* Missing keys return the **code** (dev-friendly).

### Must-have tests

* `MessagesTest`: resolves code + args via `LocaleContextHolder`.
* `ProblemKeyTest`: `titleKey/detailKey/slug` contracts hold.
* `ProblemTextsTest`: builds a valid `ProblemDetail` with default status + type URI.

---

## Phase v2 — Scale with teams/locales

### 1) Stronger structure (per domain bundles)

```
src/main/resources/i18n/
├─ messages.properties
├─ problem-messages.properties
├─ validation.properties
└─ email/
   ├─ common.properties
   └─ account.properties
```

Add locale variants when ready:

```
messages_de.properties
problem-messages_de.properties
validation_de.properties
email/common_de.properties
```

### 2) CI: Keys Consistency Gate

Add a tiny test that **fails** when:

* bundle files disagree on key sets,
* `ProblemKey` slugs don’t exist in bundles.

```java
// BundleConsistencyTest.java (fast, no Spring)
class BundleConsistencyTest {
  @Test void allBundlesHaveSameKeys() { /* load *.properties, compare key sets */ }
  @Test void problemKeysHaveTitleAndDetail() { /* for each ProblemKey */ }
}
```

### 3) Enum registry → docs & metadata

Keep `ProblemKey` as the single source of truth:

* `slug()`, `titleKey()`, `detailKey()`, optional `defaultStatus()`.
* `/problems` and `/problems/{slug}` read from the same enum + `Messages`.

### 4) Deprecation + Migration

Add optional deprecations map:

```
i18n/deprecations.properties
problem.legacy-conflict=problem.conflict
```

A tiny resolver can warn (log) and map old codes to new ones during a transition.

### 5) Dev hot-reload ergonomics

For dev profile:

```yaml
spring:
  messages:
    cache-duration: 1s
```

(Still needs classpath refresh; good enough without extra tooling.)

---

## Phase v3 — Advanced (only if needed)

### 1) ICU MessageFormat support (richer plurals)

If you need complex pluralization/gender:

* Introduce an ICU-capable MessageSource (e.g., via a small wrapper).
* Keep keys compatible (`{0, plural, one {...} other {...}}`).
* Scope to `messages*.properties` only (problems often don’t need plurals).

### 2) Extraction / linting tooling

* A tiny Gradle task or script that:

  * scans Java for `msg("…")`, `@Message("…")`, `ProblemKey.*.titleKey()`,
  * cross-checks presence in bundles,
  * emits a machine-readable report for translators.

### 3) Translation workflow (human or service)

* Freeze key sets per release tag (export a CSV: key → English text → context).
* Re-import localized files (manual PRs). Avoid database-backed messages unless you truly need runtime edits.

### 4) HTML docs for problems (already done)

* Keep `/problems` & `/problems/{slug}` as the canonical contract page.
* Optionally add a mini index page grouping problems by HTTP status.

### 5) Error contract extensions

* Add stable `code` on every `ProblemDetail` (already in your `ProblemTexts`).
* Optionally add `traceId` if you have tracing; mention it on the HTML doc page.

---

## Key Policies (short + strict)

1. **Stability:** Keys and slugs are API contracts. Add new → deprecate old; don’t silently rename.
2. **Ownership:** Each feature owns its keys; problems are owned centrally via `ProblemKey`.
3. **Consistency:** Every `ProblemKey` must have `title` + `detail` in **all** shipped locales.
4. **Reviews:** CI test blocks merges if bundles are inconsistent or missing keys.
5. **Clarity:** Messages are full sentences; placeholders are `{0}`, `{1}`, … with clear order.

---

## Ready-to-copy utilities (thin and useful)

### A. Bundle Key Linter (no Spring)

```java
// src/test/java/com/example/app/i18n/BundleKeyLinter.java
class BundleKeyLinter {
  static Properties load(String path) throws IOException {
    try (var in = ClassLoader.getSystemResourceAsStream(path)) {
      var p = new Properties(); p.load(new InputStreamReader(in, StandardCharsets.UTF_8)); return p;
    }
  }
  static Set<String> keys(String path) throws IOException { return load(path).stringPropertyNames(); }
}
```

Then use it in `BundleConsistencyTest`.

### B. ProblemKey coverage test

```java
for (var k : ProblemKey.values()) {
  assertThat(msg.tryMsg(k.titleKey())).isPresent();
  assertThat(msg.tryMsg(k.detailKey(), "X")).isPresent();
}
```

---

## “Add a new problem” SOP (copy into your vault)

1. **Enum:** add `PAYMENT_REQUIRED("payment-required", 402)` (or set `HttpStatus.PAYMENT_REQUIRED`).
2. **Bundles:** add

   ```
   problem.payment-required.title=Payment required
   problem.payment-required.detail=Payment is required to access "{0}".
   ```
3. **Docs auto-works:** `/problems/payment-required` (HTML + JSON).
4. **Handler:** controller advice returns `ProblemTexts.of(PAYMENT_REQUIRED, resourceName)`.

---

## “Add a new locale” SOP

1. Duplicate all `*.properties` to `*_xx.properties`.
2. CI will fail until key sets match.
3. Translate incrementally; PRs can add missing keys to pass the gate.

---

## “Break glass” toggles

* Dev only: `spring.messages.use-code-as-default-message=true`.
* Prod strict mode (optional): set it to `false` + fail fast in `Messages.msg()` when a key is missing (wrap with your own guard).

---

## Quick Checklist (paste into `i18n/index.md`)

* `ProblemKey` reflects all public errors (slug + keys [+ status])
* All bundles present + consistent across locales
* `/problems` + `/problems/{slug}` render from the same enum + bundles
* `Messages` used in constructors; `@Message` for frequently reused literals
* CI tests: bundle key parity, `ProblemKey` coverage, no orphaned keys
* Deprecation mapping file exists (if you’ve ever renamed keys)
* Optional ICU planned (only if you truly need advanced plurals)

---

Generate the actual **`BundleConsistencyTest`** and a tiny **Gradle task** stub to run it as part of `check`, plus a script to diff keys between locales.
