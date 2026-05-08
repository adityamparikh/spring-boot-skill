---
name: spring-boot
description: Use when creating, modifying, or working on Spring Boot projects. Triggers: creating a new Spring Boot application, adding REST endpoints, writing Spring services or repositories, configuring Spring Data JPA, writing validation, exception handling, configuration properties, observability (metrics, tracing, OpenAPI), enabling virtual threads, setting up NullAway and JSpecify null safety, writing JUnit tests for Spring components, or running a Spring Boot build, or migrating/upgrading between Spring Boot versions. Applies version-appropriate best practices, writes JavaDocs, adds JUnit tests, and runs the build. Optionally runs SonarQube analysis when the SonarQube MCP server is configured.
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
8. **Run SonarQube analysis (if available)** — when the SonarQube MCP server (`mcp__sonarqube__*`) is configured, run analysis to check for code quality violations. If it is not configured, skip this step rather than blocking the task.
9. **Fix any reported SonarQube violations** and re-run the build to confirm.

Never consider a task complete until the build passes. If a SonarQube MCP server is configured, also resolve its reported violations before completion.

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

> **For broader JSpecify topics** — migrating from other annotation libraries (JSR-305, JetBrains `org.jetbrains.annotations.*`, Spring's `org.springframework.lang.*`, Jakarta, Android, FindBugs, Checker Framework, Eclipse JDT) using OpenRewrite, incremental adoption with `@NullUnmarked`, NullAway/Error Prone enforcement details, and Kotlin interop — use the standalone `jspecify` skill. This section covers Spring-Boot-specific JSpecify usage only (which annotations to use per Boot version, package-level marking, Optional vs `@Nullable` rules).

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

---

## REST Endpoints

Build controllers thin: bind input, delegate to a service, return a response. Keep validation, persistence, and orchestration out of the controller body.

```java
@RestController
@RequestMapping("/api/orders")
@Validated
class OrderController {

    private final OrderService orders;

    OrderController(OrderService orders) {
        this.orders = orders;
    }

    @GetMapping("/{id}")
    Order getOne(@PathVariable UUID id) {
        return orders.findById(id).orElseThrow(() -> new OrderNotFoundException(id));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    Order create(@Valid @RequestBody CreateOrderRequest request) {
        return orders.create(request);
    }

    @GetMapping
    Page<Order> list(@RequestParam(defaultValue = "0") int page,
                     @RequestParam(defaultValue = "20") @Max(100) int size) {
        return orders.list(PageRequest.of(page, size));
    }
}
```

**Use records for request and response DTOs.** They are immutable, work with Bean Validation annotations on parameters, and Jackson serializes them with no extra config.

```java
record CreateOrderRequest(@NotBlank String sku,
                          @Positive int quantity,
                          @Email String customerEmail) {}
```

### API versioning

- **3.5.x**: use `@RequestMapping(headers = "X-API-Version=2")` or path prefixes (`/api/v2/...`).
- **4.0.x**: prefer Spring Framework 7's built-in `version` attribute on the mapping annotation:

  ```java
  @GetMapping(path = "/{id}", version = "2")
  Order getV2(@PathVariable UUID id) { ... }
  ```

For a full versioning strategy comparison (path / header / media-type / query), see the `spring-boot-4-migration` skill's `references/api-versioning.md`.

---

## Exception Handling and ProblemDetail

Use a single `@RestControllerAdvice` to translate domain exceptions into RFC 7807 `ProblemDetail` responses. Do not return ad-hoc error JSON.

```java
@RestControllerAdvice
class ApiExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail onNotFound(OrderNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setType(URI.create("https://example.com/problems/order-not-found"));
        pd.setProperty("orderId", ex.getOrderId());
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail onValidation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        pd.setProperty("fieldErrors",
            ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
                .toList());
        return pd;
    }
}
```

**Always:** map known domain exceptions to specific status codes. Let unmapped exceptions fall through to Spring Boot's default `ProblemDetail` handler — never catch-and-swallow with `Exception.class` unless you re-throw or add real recovery.

---

## Validation

Bean Validation 3 (Jakarta) is auto-configured by `spring-boot-starter-validation`. Apply constraints at the boundary: request DTOs and `@PathVariable` / `@RequestParam` arguments.

```java
record CreateUserRequest(
    @NotBlank @Size(max = 80) String name,
    @Email String email,
    @Past LocalDate birthDate
) {}

@RestController
@Validated  // enables constraint validation on method-level params (path/query)
class UserController {
    @GetMapping("/users/{id}")
    User get(@PathVariable @NotNull UUID id) { ... }
}
```

**Custom constraints**: implement `ConstraintValidator<MyAnnotation, T>` and register the annotation with `@Constraint(validatedBy = MyValidator.class)`. Keep validators stateless.

For cross-field rules (e.g., "endDate after startDate"), prefer a class-level constraint over scattering checks across services.

---

## Spring Data JPA

Define an entity with `@Entity`, a primary key, and explicit columns where defaults are wrong. Use `JpaRepository<T, ID>` for CRUD plus derived queries.

```java
@Entity
@Table(name = "orders")
class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private UUID id;

    @Column(nullable = false, length = 64)
    private String sku;

    @Column(nullable = false)
    private int quantity;

    @CreationTimestamp
    private Instant createdAt;

    // getters, setters, equals/hashCode by id
}

interface OrderRepository extends JpaRepository<Order, UUID> {
    List<Order> findBySkuAndQuantityGreaterThan(String sku, int quantity);

    @Query("SELECT o FROM Order o WHERE o.createdAt >= :since")
    List<Order> recent(@Param("since") Instant since);
}
```

**Transactions**: annotate write services with `@Transactional`. Read-only methods get `@Transactional(readOnly = true)` for connection-pool and Hibernate flush optimizations.

**Hibernate version differences**:
- **3.5.x**: Hibernate ORM 6.x. `GenerationType.SEQUENCE` defaults are sane; `IDENTITY` blocks JDBC batching.
- **4.0.x**: Hibernate ORM 7.1.x. Some entity-mapping defaults tighten; review `references/spring-framework7.md` in the `spring-boot-4-migration` skill before upgrading.

For deep JPA review (N+1 detection, fetch plans, batch tuning, connection pooling, query count assertions), use the `hibernate-jpa-validator` skill — it activates on JPA-specific code review and will not duplicate guidance here.

---

## Configuration Properties

Use type-safe `@ConfigurationProperties` over scattered `@Value` injection. Records work great for immutable config.

```java
@ConfigurationProperties("app.payments")
@Validated
record PaymentsProperties(
    @NotBlank String apiKey,
    @NotNull URI endpoint,
    @DefaultValue("PT5S") Duration timeout,
    Retry retry
) {
    record Retry(@DefaultValue("3") @Min(0) int maxAttempts,
                 @DefaultValue("PT200MS") Duration backoff) {}
}
```

Register with `@EnableConfigurationProperties(PaymentsProperties.class)` on a `@Configuration` class, or annotate the record with `@ConfigurationPropertiesScan` at the application level.

**Version differences for `@ConfigurationProperties`**:
- **3.5.x**: records, classes with constructor binding, and classes with public-field binding all work.
- **4.0.x**: public-field binding is **removed**. Use records or classes with private fields and getters/setters.

Profiles: keep environment-specific overrides in `application-{profile}.yaml` (or `.properties`). Activate with `SPRING_PROFILES_ACTIVE=prod` or `--spring.profiles.active=prod`. Avoid mixing profile config into the base file.

---

## Testing Spring Components

Match the test slice to what you are testing. Full-context tests are slow — use them only when integration is the point.

| Slice | Annotation | Loads | Use when |
|---|---|---|---|
| Web layer only | `@WebMvcTest(MyController.class)` | controller, advice, MVC infrastructure | testing request mapping, validation, serialization |
| JPA layer only | `@DataJpaTest` | entities, repositories, in-memory DB or Testcontainers | testing repository queries, mappings |
| Full app | `@SpringBootTest` | everything | wiring/integration tests against a real HTTP port |

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvcTester mvc;        // 3.4+ fluent API; preferred over MockMvc
    @MockitoBean OrderService orders;    // 3.4+ replacement for @MockBean

    @Test
    void returnsCreatedOnPost() {
        when(orders.create(any())).thenReturn(new Order(/* ... */));

        mvc.post().uri("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"sku":"ABC","quantity":2,"customerEmail":"a@b.com"}
                """)
            .assertThat()
            .hasStatus(HttpStatus.CREATED)
            .bodyJson()
            .hasPathSatisfying("$.sku", v -> v.assertThat().isEqualTo("ABC"));
    }
}
```

**Mock annotations by version**:
- **3.5.x**: prefer `@MockitoBean` / `@MockitoSpyBean` (introduced in 3.4). `@MockBean` / `@SpyBean` still work but are deprecated.
- **4.0.x**: `@MockBean` and `@SpyBean` are **removed**. Use `@MockitoBean` / `@MockitoSpyBean`.

**HTTP test clients**:
- 3.5.x: `MockMvc`, `MockMvcTester` (3.4+), or `WebTestClient` (WebFlux).
- 4.0.x: same plus `RestTestClient` for `RestClient`-style assertions in `@SpringBootTest`.

**JUnit version**:
- 3.5.x: JUnit 5 (Jupiter).
- 4.0.x: JUnit 6.

**Testcontainers**: use `@Testcontainers` + `@ServiceConnection` (Boot 3.1+) so Spring Boot wires the container's connection details directly into the test context — no manual `@DynamicPropertySource` needed.

```java
@SpringBootTest
@Testcontainers
class OrderIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired OrderRepository repository;

    @Test
    void persistsAndQueries() { /* ... */ }
}
```

For reactive testing (`StepVerifier`, `WebTestClient`, virtual time), use the `project-reactor` skill.

---

## Frontmatter Reference

This skill follows the [official Claude Code Skill spec](https://code.claude.com/docs/en/skills#frontmatter-reference). The `allowed-tools` field uses a YAML list of tool names with permission patterns (`Bash`, `Bash(git status *)`, `mcp__server__*`, etc.). The `argument-hint` field was removed because this skill is auto-invoked from the description and does not consume `$ARGUMENTS` in its body.
