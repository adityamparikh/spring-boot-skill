# Code Organization

Default: **package by feature, not by layer.** Spring Boot ships no opinion here, so be deliberate.

## Why feature-first beats layer-first

Layered packages (`controller/`, `service/`, `repository/`, `model/`) optimize for "find every controller" -- a question almost nobody asks. They scatter every change across four packages and force every internal collaborator to be `public`, which makes encapsulation impossible.

Feature packages co-locate the controller, service, repository, domain types, and DTOs for one bounded concern. Benefits:

- Most changes touch one package.
- Cross-feature collaborators must go through a small, deliberate API surface (use package-private types for everything that is not the public contract).
- Deletion is cheap: removing a feature is removing a directory.
- Onboarding follows the business, not the framework.

## Default layout

```
com.example.app
├── Application.java
├── orders/                       # feature
│   ├── package-info.java         # @NullMarked, @ApplicationModule(...)
│   ├── OrderController.java      # public-facing API surface
│   ├── OrderService.java         # package-private impl
│   ├── OrderRepository.java      # package-private
│   ├── Order.java                # entity / record
│   ├── OrderDto.java
│   └── internal/                 # optional: deeper internals
├── shipping/
│   └── ...
└── shared/                       # cross-cutting only; resist growth
    └── ...
```

Rules:

- Feature packages are siblings under the application root.
- Default visibility is **package-private**. Mark a type `public` only when another module legitimately needs it.
- Cross-feature calls go through events (preferred) or a narrow published API. Direct repository-to-repository calls across features are a smell.
- A `shared/` package is a last resort. If something keeps moving in, it is probably its own feature.

## Spring Modulith

For applications with more than ~3 features, adopt **Spring Modulith** to make module boundaries explicit, verifiable, and observable. It treats top-level packages as application modules and gives runtime introspection, build-time verification, and module-aware test slices.

### Dependencies

```
org.springframework.modulith:spring-modulith-starter-core         # always
org.springframework.modulith:spring-modulith-starter-test         # tests
org.springframework.modulith:spring-modulith-events-jpa           # event publication registry on JPA
org.springframework.modulith:spring-modulith-events-kafka         # externalize events to Kafka (or amqp/jms variants)
org.springframework.modulith:spring-modulith-observability        # module-level traces/metrics
org.springframework.modulith:spring-modulith-actuator             # /actuator/modulith
```

Use the Spring Boot BOM-managed Modulith version. Verify the latest with Context7 / release notes.

### Modules

Each top-level package under the `@SpringBootApplication` class is an application module. Annotate `package-info.java` to make it explicit and document allowed dependencies:

```java
@org.springframework.modulith.ApplicationModule(
    displayName = "Orders",
    allowedDependencies = {"shipping", "payments"}
)
@org.jspecify.annotations.NullMarked
package com.example.app.orders;
```

Anything not in `allowedDependencies` becomes a verification failure. Empty (`{}`) means "this module talks to no other module."

### Named interfaces

Sub-packages are internal to the module by default. To expose more than one entry point, declare named interfaces:

```java
@org.springframework.modulith.NamedInterface("api")
package com.example.app.orders.api;
```

Other modules then depend on the named interface, not the module root.

### Inter-module communication: events first

Prefer Spring application events over direct calls between modules:

```java
@org.springframework.modulith.events.ApplicationModuleListener
void on(OrderPlaced event) { ... }
```

`@ApplicationModuleListener` is `@Async` + `@Transactional(propagation = REQUIRES_NEW)` + `@TransactionalEventListener(AFTER_COMMIT)`. It is the right default for cross-module workflows.

For at-least-once delivery across restarts, enable the **event publication registry** (`spring-modulith-events-jpa` writes pending publications to a table; failed listeners are retried on startup). For inter-service delivery, annotate events `@Externalized` and add the broker starter.

### Verification

Add one test that fails the build if module rules are violated:

```java
@org.junit.jupiter.api.Test
void modulesAreClean() {
    org.springframework.modulith.core.ApplicationModules.of(Application.class).verify();
}
```

Optionally render the module graph for docs:

```java
var modules = ApplicationModules.of(Application.class);
new org.springframework.modulith.docs.Documenter(modules)
    .writeModulesAsPlantUml()
    .writeIndividualModulesAsPlantUml();
```

### Test slicing

`@ApplicationModuleTest` boots only the target module and stubs the rest. Combine with `Scenario` for event-driven assertions:

```java
@org.springframework.modulith.test.ApplicationModuleTest
class OrdersModuleTests {

    @Test
    void publishesOrderPlaced(Scenario scenario) {
        scenario.stimulate(() -> orders.place(...))
            .andWaitForEventOfType(OrderPlaced.class)
            .matching(e -> e.id().equals(...))
            .toArrive();
    }
}
```

### Observability

Adding `spring-modulith-observability` instruments inter-module calls and event listeners with Micrometer / OpenTelemetry. Combined with `spring-modulith-actuator`, the live module structure is exposed at `/actuator/modulith`.

## When NOT to modulith

- Single-feature service (one CRUD aggregate) -- feature-package layout is enough; the dependency is overhead.
- Code that is already a microservice -- Modulith is for monoliths that want module discipline; do not layer it onto an already-decomposed system unless you are consolidating.

## Migrating from layered to feature

1. Pick one vertical slice (e.g. orders).
2. Create `orders/` and move its controller, service, repository, entities into it.
3. Drop the now-unnecessary `public` modifiers; rely on package-private.
4. Repeat per feature.
5. Once 2-3 features exist as packages, add `spring-modulith-starter-core` and the verification test, then start annotating `package-info.java` files.
6. Introduce events before allowing direct cross-module calls.

## Never do

- Do not create a global `dto/`, `service/`, or `controller/` package alongside feature packages -- it pulls you back into layering.
- Do not make every type `public` "in case." Public is a contract, not a default.
- Do not put cross-feature business logic in `shared/`. That is what events and named interfaces are for.
- Do not skip the `ApplicationModules.verify()` test -- without it, modulith annotations decay into documentation.
