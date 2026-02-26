---
name: spring-boot
description: Use when creating, modifying, or working on Spring Boot projects. Triggers: creating a new Spring Boot application, adding REST endpoints, writing Spring services or repositories, configuring Spring Data JPA, adding observability (metrics, tracing, OpenAPI), enabling virtual threads, setting up NullAway and JSpecify null safety, writing JUnit tests for Spring components, or running a Spring Boot build, or migrating/upgrading between Spring Boot versions. Applies version-appropriate best practices, writes JavaDocs, adds JUnit tests, runs the build, and checks SonarQube for violations.
allowed-tools: Bash(*), WebFetch(*), WebSearch(*), mcp__sonarqube__*, mcp__claude_ai_Context7__*, mcp__plugin_context7_context7__*
argument-hint: "[description of what to build or change]"
---

# Spring Boot Development Skill

## Version Detection

Before making any changes, detect the project's Spring Boot and Java versions:

1. **Check the build file** (`pom.xml` or `build.gradle`/`build.gradle.kts`) for:
   - `spring-boot-starter-parent` version or Spring Boot plugin version
   - `java.version` / `sourceCompatibility` / `targetCompatibility`
2. **Apply version-appropriate code** using the compatibility reference below.

### Supported Versions

| Spring Boot | Spring Framework | Jakarta EE | Java | Jackson | JUnit |
|-------------|-----------------|------------|------|---------|-------|
| 3.5.x | 6.x | 9/10 | 17, 21 | 2.x | 5 (Jupiter) |
| 4.0.x | 7.x | 11 | 17, 21, 25 | 3.x | 6 |

### Version-Specific Differences

**Spring Boot 4.x (Spring Framework 7 / Jakarta EE 11) -- Java 21+ recommended, virtual threads enabled by default:**
- **Servlet containers:** Requires Tomcat 11+ or Jetty 12.1+. Undertow is NOT supported.
- **Null safety:** Use JSpecify annotations (`org.jspecify.annotations.Nullable`, `@NullMarked`) instead of `org.springframework.lang` annotations.
- **API versioning:** Use built-in native versioning on `@GetMapping`/`@PostMapping` etc. with `version` attribute -- no custom `RequestMappingHandlerMapping` needed.
- **Resilience:** Use built-in `@Retryable` and `@ConcurrencyLimit` from Spring Framework 7 instead of adding Spring Retry separately.
- **Configuration properties:** Public field binding is removed. Use private fields with getters/setters for `@ConfigurationProperties`.
- **Testing:** `@MockBean` and `@SpyBean` are removed. Use the replacements introduced in 3.5.x.
- **RestTestClient:** Use `RestTestClient` for testing HTTP endpoints (autowire it in `@SpringBootTest` or `@AutoConfigureMockMvc` tests).
- **Jackson 3.x:** JSON processing uses Jackson 3. Verify any custom serializers/deserializers are compatible.
- **JUnit 6:** Tests use JUnit 6. Ensure test annotations and APIs match.
- **Modular autoconfigure:** Autoconfiguration JARs are split into focused modules.
- **Commons Logging:** `spring-jcl` is removed; Apache Commons Logging 1.3.0 is used directly.
- **Path matching:** `AntPathMatcher` for HTTP request mappings is deprecated; use `PathPattern` instead.

**Spring Boot 3.5.x (Spring Framework 6.x / Jakarta EE 9/10):**
- **Servlet containers:** Tomcat 10+, Jetty 12, Undertow supported.
- **Null safety:** Use `org.springframework.lang.Nullable` / `@NonNull`.
- **Testing:** `@MockBean` and `@SpyBean` still available (deprecated). Prefer the new replacements.
- **Jackson 2.x:** Standard Jackson 2.x APIs.
- **JUnit 5 (Jupiter):** Standard JUnit 5 APIs.
- **Spring Retry:** Add `spring-retry` dependency separately for `@Retryable`.
- **Virtual threads:** Supported on Java 21+ with `spring.threads.virtual.enabled=true` (available since 3.2). Not available on Java 17.

### New Project Creation

When creating a new project, default to:
- **Spring Boot 4.0.x** (latest stable)
- **Java 21** (recommended LTS)

Unless the user specifies otherwise.

## Verify Library and Framework Usage

The model's training data has a knowledge cutoff. When working with Spring Boot projects, **actively verify** that APIs, configurations, and patterns are current for the detected version:

1. **Identify versions** — Check `pom.xml` or `build.gradle.kts` to determine the exact versions of Spring Boot, Spring Framework, and all key dependencies.
2. **Look up current documentation** — Use Context7 (`mcp__claude_ai_Context7__resolve-library-id` then `mcp__claude_ai_Context7__query-docs`) to retrieve up-to-date documentation for any library or framework where:
   - The version is newer than what the model may have been trained on
   - You are unsure whether an API, configuration property, or pattern is still valid or has been deprecated/replaced
   - The code uses advanced or less common features of a library
3. **Search the web** — Use `WebSearch` and `WebFetch` to check for:
   - Breaking changes or migration guides for the specific version in use
   - Known security vulnerabilities (CVEs) in the dependency versions
   - Deprecated APIs that have been replaced in newer versions
4. **Check GitHub** — Use `gh` CLI to check release notes, changelogs, or issues for dependencies when needed (e.g., `gh api repos/{owner}/{repo}/releases/latest`)

**Do not assume** that an API or pattern is correct based solely on model knowledge. When in doubt, look it up.

## Workflow

For every code change, follow this sequence:

1. **Detect version** -- read the build file to determine Spring Boot and Java versions
2. **Understand** the request and identify affected files/components
3. **Implement** the change following the guidelines below, applying version-appropriate code
4. **Write JavaDocs** for all public and protected members
5. **Write or update tests** for every code change
6. **Run the build** to verify compilation and tests pass
7. **Fix any build failures** before proceeding
8. **Run SonarQube analysis** using the SonarQube MCP server tools to check for code quality violations
9. **Fix all SonarQube violations** and re-run the build to confirm

Never consider a task complete until the build passes and SonarQube violations are resolved.

---

## Virtual Threads

Enable virtual threads by default for all new and existing projects running on **Java 21+**.

#### Coding Rules When Virtual Threads Are Enabled
- **Never pool virtual threads.** Use `Executors.newVirtualThreadPerTaskExecutor()` for custom executors.
- **Use `ReentrantLock` instead of `synchronized`** for blocking I/O — `synchronized` pins the carrier thread.
- **Avoid `ThreadLocal` for large or long-lived state.** Use `ScopedValue` (Java 21+ preview / Java 25 stable) or pass context explicitly.
- **Do not set thread priorities or daemon status.** Virtual threads ignore both.
- **Remove manual thread pools for I/O tasks.** The virtual thread scheduler handles this automatically.

---

## Null Safety (JSpecify)

Apply null safety annotations consistently across the codebase. The annotation style depends on the Spring Boot version.

### Dependency

For Spring Boot 4.x, JSpecify is already included transitively. For Spring Boot 3.5.x, add it explicitly:

**Maven:**
```xml
<dependency>
    <groupId>org.jspecify</groupId>
    <artifactId>jspecify</artifactId>
    <version>1.0.0</version>
</dependency>
```

**Gradle:**
```groovy
implementation 'org.jspecify:jspecify:1.0.0'
```

### Which Annotations to Use

| Spring Boot | Package | Annotations |
|-------------|---------|-------------|
| 4.x | `org.jspecify.annotations` | `@Nullable`, `@NonNull`, `@NullMarked`, `@NullUnmarked` |
| 3.5.x | `org.jspecify.annotations` (preferred for forward-compat) or `org.springframework.lang` | `@Nullable`, `@NonNull`, `@NullMarked` |

Always prefer JSpecify annotations even on 3.5.x to ease future migration to 4.x.

### Applying `@NullMarked`

Mark entire packages as non-null by default using `@NullMarked` in `package-info.java`:

```java
@NullMarked
package com.example.orders;

import org.jspecify.annotations.NullMarked;
```

With `@NullMarked`, every parameter, return type, and field is assumed non-null unless explicitly annotated with `@Nullable`. This eliminates the need to annotate every non-null member individually.

Create a `package-info.java` in every package of the project.

### Using `@Nullable`

Only annotate members that legitimately accept or return null.

### Rules

- **Prefer `Optional<T>` over `@Nullable`** for method return types, especially in service and repository layers. Use `@Nullable` for fields, parameters, and cases where `Optional` is inappropriate (e.g., entity fields, DTO records, performance-sensitive internals).
- **Never pass `null` to a non-null parameter.** The `@NullMarked` package declaration means every unannotated parameter rejects null. Validate inputs at system boundaries instead.
- **Do not use `@NonNull` inside `@NullMarked` packages.** It is redundant -- everything is already non-null by default. Only use `@NonNull` in packages that are not `@NullMarked`.
- **Use `@NullUnmarked`** sparingly to opt out specific classes or methods that interact with legacy code where nullability is unknown. Document why.
- **Annotate generic type arguments** when nullability varies:

```java
// The list itself is non-null, but elements may be null
List<@Nullable String> parseOptionalFields(String input);
```

For NullAway/Error Prone build configuration, read `references/nullaway-setup.md`.

---

## Observability

For OpenAPI (springdoc-openapi), Micrometer metrics, and distributed tracing setup, read `references/observability.md`.
