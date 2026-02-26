# Observability Setup

## OpenAPI Documentation (springdoc-openapi)

### Setup
- Use `springdoc-openapi` v2.x (required for Spring Boot 3). Never use v1.x or SpringFox.
- Add the appropriate starter dependency:
  - Web MVC with Swagger UI: `springdoc-openapi-starter-webmvc-ui`
  - Web MVC API only: `springdoc-openapi-starter-webmvc-api`
  - WebFlux: `springdoc-openapi-starter-webflux-api`
- Use the `springdoc-openapi-bom` for consistent version management across modules.

### API Grouping
- Define `GroupedOpenApi` beans to organize endpoints by domain or version:

```java
@Bean
GroupedOpenApi ordersApi() {
    return GroupedOpenApi.builder()
            .group("orders")
            .pathsToMatch("/api/v1/orders/**")
            .build();
}
```

### Configuration
- Set `springdoc.api-docs.path` and `springdoc.swagger-ui.path` as needed.
- **Disable in production:** Set `springdoc.api-docs.enabled=false` and `springdoc.swagger-ui.enabled=false` in production profiles.
- For applications behind a reverse proxy, configure `server.forward-headers-strategy=framework`.

### Build-Time Generation
- **Maven:** Use `springdoc-openapi-maven-plugin` to generate the OpenAPI spec (JSON/YAML) during the integration-test phase.
- **Gradle:** Use the `org.springdoc.openapi-gradle-plugin` plugin to generate the spec during the build.

---

## Metrics (Micrometer)

### Setup
- Include `spring-boot-starter-actuator` (provides Micrometer auto-configuration).
- Add the appropriate registry dependency for your backend:
  - Prometheus: `micrometer-registry-prometheus`
  - OTLP: `micrometer-registry-otlp`
  - Datadog, New Relic, etc.: corresponding `micrometer-registry-*` dependency.

### Tags and Cardinality
- Restrict tag values to a known, finite set.

### Security
- Expose `/actuator/prometheus` (or equivalent) only to your metrics scraper, not publicly.
- Configure via `management.endpoints.web.exposure.include` and Spring Security.

---

## Distributed Tracing (Micrometer Tracing + OpenTelemetry)

### Setup
Spring Cloud Sleuth is deprecated in Spring Boot 3. Use Micrometer Tracing instead.

Add these dependencies:
- `spring-boot-starter-actuator`
- `io.micrometer:micrometer-tracing-bridge-otel` (bridges Micrometer to OpenTelemetry)
- `io.opentelemetry:opentelemetry-exporter-otlp` (exports traces via OTLP)

Configure the exporter endpoint:
```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # 1.0 for dev, 0.1 for production
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
```

### Context Propagation
- W3C Trace Context headers (`traceparent`, `tracestate`) are automatically propagated.
- For `@Async` methods and custom thread pools, use `ContextExecutorService` or `TaskDecorator` to propagate tracing context across threads.
- If an API gateway terminates/creates traces, ensure it forwards `traceparent`/`tracestate` headers downstream.

### Log Correlation
- Include `traceId` and `spanId` in log patterns:

```xml
<!-- logback-spring.xml -->
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{traceId}/%X{spanId}] %-5level %logger{36} - %msg%n</pattern>
```

### `@Observed` for Declarative Observability
- Annotate methods with `@Observed` (from Micrometer) to auto-generate both a trace span and a timer metric.
- Register an `ObservedAspect` bean in your configuration to activate `@Observed`.

### Sampling
- Use `management.tracing.sampling.probability` to control the sampling rate.
- Set `1.0` (100%) in development, `0.1` (10%) or lower in production to reduce overhead.

### Production Architecture
- Deploy an OpenTelemetry Collector as a sidecar or standalone service.
- Route: **App -> OTLP -> OTel Collector -> Backend** (Tempo, Jaeger, Zipkin, etc.).
- Never export traces directly to the storage backend from the application in production.
