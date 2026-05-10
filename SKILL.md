---
name: spring-boot
description: Use when creating, modifying, or working on Spring Boot projects. Triggers: creating a new Spring Boot application, adding REST endpoints, writing Spring services or repositories, configuring Spring Data JPA, writing validation, exception handling, configuration properties, observability (metrics, tracing, OpenAPI), enabling virtual threads, setting up NullAway and JSpecify null safety, writing JUnit tests for Spring components, or running a Spring Boot build. Applies version-appropriate best practices (Boot 3.5.x and 4.0.x), writes JavaDocs, adds JUnit tests, and runs the build. Do NOT use for: full Spring Boot major-version migrations (use the spring-boot-4-migration skill instead), pure JPA/Hibernate code review (use hibernate-jpa-validator), broader JSpecify migration across non-Spring annotation libraries (use the jspecify skill), or reactive testing details (use project-reactor).
allowed-tools:
  - Bash
  - WebFetch
  - WebSearch
  - mcp__sonarqube__*
  - mcp__claude_ai_Context7__*
  - mcp__plugin_context7_context7__*
---

# Spring Boot Development Skill

## Version Detection

Before making any changes, detect the project's Spring Boot and Java versions:

1. Read `pom.xml` or `build.gradle`/`build.gradle.kts` for `spring-boot-starter-parent` (or the Spring Boot plugin) version and `java.version` / `sourceCompatibility` / `targetCompatibility`.
2. Apply version-appropriate code using the table below.

### Supported Versions

| Spring Boot | Spring Framework | Jakarta EE | Java | Jackson | JUnit |
|-------------|-----------------|------------|------|---------|-------|
| 3.5.x | 6.x | 9/10 | 17, 21 | 2.x | 5 (Jupiter) |
| 4.0.x | 7.x | 11 | 17, 21, 25 | 3.x | 6 |

### Version-Specific Differences

**Spring Boot 4.x (Spring Framework 7 / Jakarta EE 11) -- Java 21+ recommended, virtual threads enabled by default:**
- Servlet containers: Tomcat 11+ or Jetty 12.1+. Undertow is NOT supported.
- Null safety: JSpecify (`org.jspecify.annotations.*`) only.
- API versioning: built-in `version` attribute on `@GetMapping`/`@PostMapping`.
- Resilience: built-in `@Retryable` and `@ConcurrencyLimit` from Spring Framework 7.
- `@ConfigurationProperties`: public-field binding removed -- use records or private fields with getters/setters.
- Testing: `@MockBean`/`@SpyBean` removed -- use `@MockitoBean`/`@MockitoSpyBean`. `RestTestClient` available.
- Jackson 3.x, JUnit 6, modular autoconfigure, `spring-jcl` removed (Commons Logging 1.3.0 used directly).
- `AntPathMatcher` for HTTP request mappings deprecated -- use `PathPattern`.

**Spring Boot 3.5.x (Spring Framework 6.x / Jakarta EE 9/10):**
- Servlet containers: Tomcat 10+, Jetty 12, Undertow.
- Null safety: prefer `org.jspecify.annotations.*` (forward-compat) over `org.springframework.lang.*`.
- Testing: `@MockBean`/`@SpyBean` deprecated -- prefer `@MockitoBean`/`@MockitoSpyBean` (3.4+).
- Jackson 2.x, JUnit 5. Spring Retry requires the `spring-retry` dependency.
- Virtual threads on Java 21+ via `spring.threads.virtual.enabled=true`.

### New Project Defaults

Default to Spring Boot 4.0.x and Java 21 unless the user specifies otherwise.

## Verify Library and Framework Usage

The model's training data has a knowledge cutoff. Actively verify APIs and configuration are current for the detected version:

1. Identify exact dependency versions from the build file.
2. Use Context7 (`mcp__claude_ai_Context7__resolve-library-id` then `query-docs`) for up-to-date docs when the version is newer than training, or you are unsure whether an API is still valid.
3. Use `WebSearch`/`WebFetch` for breaking changes, migration guides, and CVEs.
4. Use `gh` to check release notes when needed.

Do not assume APIs based on model knowledge. When in doubt, look it up.

## Workflow

For every code change:

1. Detect Spring Boot and Java versions.
2. Identify affected files/components.
3. Implement using version-appropriate code.
4. Write JavaDocs for public and protected members.
5. Write or update tests.
6. Run the build.
7. Fix any build failures.
8. If the SonarQube MCP server (`mcp__sonarqube__*`) is configured, run analysis. Skip silently if not configured.
9. Fix any SonarQube violations and re-run the build.

Never consider a task complete until the build passes (and SonarQube violations are resolved when applicable).

## Topic-Specific Guidance

Load the relevant spoke file on demand for the task at hand:

| Task | Read |
|------|------|
| Building controllers, request/response DTOs, API versioning | `references/rest-controllers.md` |
| `@RestControllerAdvice`, `ProblemDetail`, RFC 7807 errors | `references/exception-handling.md` |
| Bean Validation 3 (Jakarta), custom constraints | `references/validation.md` |
| `@Entity`, `JpaRepository`, `@Transactional`, Hibernate notes | `references/spring-data-jpa.md` |
| `@ConfigurationProperties`, profiles | `references/configuration-properties.md` |
| `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`, Testcontainers, mock-bean annotations | `references/testing.md` |
| OpenAPI, Micrometer metrics, distributed tracing | `references/observability.md` |
| NullAway / Error Prone build configuration | `references/nullaway-setup.md` |

## Virtual Threads

Enable by default on Java 21+ (`spring.threads.virtual.enabled=true`). When enabled:

- Never pool virtual threads. Use `Executors.newVirtualThreadPerTaskExecutor()` for custom executors.
- Use `ReentrantLock` instead of `synchronized` around blocking I/O (`synchronized` pins the carrier thread).
- Avoid `ThreadLocal` for large or long-lived state. Prefer `ScopedValue` (preview on 21, stable on 25) or pass context explicitly.
- Do not set thread priorities or daemon status -- virtual threads ignore both.
- Remove manual thread pools for I/O tasks; the virtual thread scheduler handles them.

## Null Safety (JSpecify)

For broader JSpecify topics (migrating from JSR-305, JetBrains, Spring's `org.springframework.lang.*`, Jakarta, Android, FindBugs, Checker, Eclipse JDT via OpenRewrite; incremental adoption with `@NullUnmarked`; Kotlin interop) use the standalone `jspecify` skill. This section covers Spring-Boot-specific usage only.

### Dependency

Spring Boot 4.x includes JSpecify transitively. For 3.5.x, add it explicitly:

- Maven: `org.jspecify:jspecify:1.0.0`
- Gradle: `implementation 'org.jspecify:jspecify:1.0.0'`

### Annotations to Use

| Spring Boot | Package | Annotations |
|-------------|---------|-------------|
| 4.x | `org.jspecify.annotations` | `@Nullable`, `@NonNull`, `@NullMarked`, `@NullUnmarked` |
| 3.5.x | `org.jspecify.annotations` (preferred for forward-compat) or `org.springframework.lang` | `@Nullable`, `@NonNull`, `@NullMarked` |

Prefer JSpecify on 3.5.x to ease the 4.x upgrade.

### Rules

- Mark every package non-null by default with `@NullMarked` in `package-info.java`. Create one in every package.
- Only annotate members that legitimately accept or return null with `@Nullable`.
- Prefer `Optional<T>` over `@Nullable` for service/repository return types. Use `@Nullable` for fields, parameters, and entity/DTO records where `Optional` is inappropriate.
- Never pass `null` to a non-null parameter. Validate inputs at system boundaries.
- Inside `@NullMarked` packages, do not use `@NonNull` -- it is redundant.
- Use `@NullUnmarked` sparingly to interop with legacy code, and document why.
- Annotate generic type arguments when nullability varies, e.g. `List<@Nullable String>`.

For NullAway/Error Prone build configuration, read `references/nullaway-setup.md`.
