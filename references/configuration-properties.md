# Configuration Properties

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

## Version Differences

- **3.5.x**: records, classes with constructor binding, and classes with public-field binding all work.
- **4.0.x**: public-field binding is **removed**. Use records or classes with private fields and getters/setters.

## Profiles

Keep environment-specific overrides in `application-{profile}.yaml` (or `.properties`). Activate with `SPRING_PROFILES_ACTIVE=prod` or `--spring.profiles.active=prod`. Avoid mixing profile config into the base file.
