# Asynchronous Execution: `@Async`, `ThreadPoolTaskExecutor`, and `CompletableFuture`

Companion reference to the *Asynchronous Execution* section of `SKILL.md`. Covers the production-quality template for `ThreadPoolTaskExecutor`, the `TaskDecorator` that propagates MDC / Security / Tracing context, async exception handling, and migration to virtual threads.

For pure-JVM concurrency theory (Goetz pool-sizing formula, `CompletableFuture` API, race-bug heuristics, lock profiling, virtual-thread mechanics) see the `java-performance` skill's `references/concurrency-and-threads.md` and `references/virtual-threads.md`.

---

## When to use `@Async`

| Situation | Use `@Async`? |
|---|---|
| Fire-and-forget side effect on a request (email, audit log to slow store) | ✅ Yes — return `void` and rely on the executor for back-pressure |
| Need a result later, called from a controller | ✅ Yes — return `CompletableFuture<T>` |
| You already have a non-blocking call (`WebClient`, `Mono`, reactive repository) | ❌ No — it's already async; wrapping adds a worker thread for nothing |
| CPU-bound work you want parallelised | ❌ No — use a dedicated executor or `parallelStream` with a bounded pool |
| Anything inside a `@Transactional` boundary that needs to share the transaction | ❌ No — `@Async` always runs on another thread; the transaction does not propagate |

Enable with `@EnableAsync` on a `@Configuration` class.

---

## The four `@Async` pitfalls

### 1. Self-call bypass

Spring's `@Async` (like `@Transactional`) is implemented as a CGLIB/JDK proxy. Calls **within the same bean** go straight to the method without proxy interception — the asynchronous behaviour silently disappears.

```java
@Service
class ReportService {

    public void run() {
        sendReport();  // ❌ runs synchronously on the caller thread
    }

    @Async
    public void sendReport() { ... }
}
```

Fixes:
- Inject `self` lazily: `@Lazy ReportService self;` then call `self.sendReport()`.
- Move `sendReport()` to a separate bean (`@Service ReportSender { @Async send() }`).

### 2. ThreadLocal context loss

Anything stored in a `ThreadLocal` does **not** cross to the executor's worker thread:

- `SecurityContextHolder` (Spring Security)
- `RequestContextHolder` (request/session scoped beans, MDC, `LocaleContextHolder`)
- `TransactionSynchronizationManager` (current transaction status)
- `MDC` (Logback / Log4j2 mapped diagnostic context)
- `Observation` / `Tracer` (Micrometer Tracing — see also Reactor Context for the reactive equivalent)

Fix with a `TaskDecorator` on the executor (see template below).

### 3. Transaction loss

The `@Async` method does **not** inherit the caller's `@Transactional` boundary. If it touches the DB, it needs its own `@Transactional` annotation and will acquire its own JDBC connection from the pool. Important consequences:

- A `@Transactional` method that fans out N parallel `@Async` calls consumes **N + 1** connections, not 1. Watch HikariCP `hikaricp_connections_acquire_seconds` and pool size accordingly.
- The outer transaction can commit before the async work runs. If you need the async work to be triggered *only* after commit, prefer `@TransactionalEventListener(phase = AFTER_COMMIT)` over `@Async`.
- Errors in the async work do **not** roll back the outer transaction. Compensating actions are your responsibility.

### 4. Silent exception loss

| `@Async` return type | What happens on exception |
|---|---|
| `void` | Caller never sees it. Spring calls the registered `AsyncUncaughtExceptionHandler` (default: just logs). Configure one to forward to Sentry / metrics / etc. |
| `Future<T>` / `CompletableFuture<T>` | Exception is stored in the future. **Caller must call `.get()`, `.exceptionally()`, `.handle()`, or `.whenComplete()` or it disappears.** |

Custom handler:

```java
@Configuration
@EnableAsync
class AsyncConfig implements AsyncConfigurer {
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Uncaught async exception in {}", method, ex);
            // forward to Sentry, Micrometer counter, etc.
        };
    }
}
```

---

## `ThreadPoolTaskExecutor` — production template

```java
@Configuration
@EnableAsync
class ExecutorConfig {

    @Bean(name = "ordersExecutor")
    ThreadPoolTaskExecutor ordersExecutor(
            @Value("${app.orders.executor.core-size:8}")  int coreSize,
            @Value("${app.orders.executor.max-size:25}")   int maxSize,
            @Value("${app.orders.executor.queue:100}")     int queueCapacity) {

        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(coreSize);
        executor.setMaxPoolSize(maxSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix("orders-");
        executor.setTaskDecorator(new ContextPropagatingTaskDecorator());
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }
}
```

Key choices and why:

| Setting | Choice | Reason |
|---|---|---|
| `corePoolSize` | Configurable, default = number of expected concurrent units | Tune per workload; never hard-code |
| `maxPoolSize` | Capped (here 25) | Prevents runaway growth on burst |
| `queueCapacity` | Bounded (here 100) | An unbounded queue defeats `maxPoolSize` — threads only grow once the queue is full |
| `RejectedExecutionHandler` | `CallerRunsPolicy` | Back-pressure on the caller instead of silently dropping (`DiscardOldestPolicy`) or throwing (`AbortPolicy`) |
| `WaitForTasksToCompleteOnShutdown` | `true` | Graceful shutdown — finish in-flight work |
| `AwaitTerminationSeconds` | 30 | Bounded wait so a hung task doesn't block the JVM exit |
| `ThreadNamePrefix` | Workload-specific | Thread dumps and JFR are unreadable without this |
| `TaskDecorator` | Custom | Propagates MDC, Security, Tracing — see below |

**Use one executor per workload class.** Mixing CPU-bound and IO-bound tasks on the same pool causes the IO tasks to starve the CPU tasks (or vice versa). Name the executor (`@Bean(name = "...")`) and reference it explicitly: `@Async("ordersExecutor")`.

**Never use `Executors.newCachedThreadPool()` or `Executors.newFixedThreadPool(N)` for production work.** The first has unbounded thread growth; the second has an unbounded queue. `ThreadPoolTaskExecutor` (or `new ThreadPoolExecutor(...)` directly) with explicit bounds is the rule.

---

## `TaskDecorator` — propagating context across threads

```java
class ContextPropagatingTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        // Capture context on the submitting thread
        Map<String, String> mdc                = MDC.getCopyOfContextMap();
        SecurityContext securityContext        = SecurityContextHolder.getContext();
        RequestAttributes requestAttributes    = RequestContextHolder.getRequestAttributes();

        return () -> {
            // Restore on the worker thread
            try {
                if (mdc != null) MDC.setContextMap(mdc);
                SecurityContextHolder.setContext(securityContext);
                if (requestAttributes != null) {
                    RequestContextHolder.setRequestAttributes(requestAttributes);
                }
                runnable.run();
            } finally {
                MDC.clear();
                SecurityContextHolder.clearContext();
                RequestContextHolder.resetRequestAttributes();
            }
        };
    }
}
```

For Micrometer Observation / tracing context, prefer the framework-provided helper:

```java
executor.setTaskDecorator(new ContextPropagatingTaskDecorator());  // Spring 6.1+ ships this
// or, explicit Micrometer:
executor.setTaskDecorator(new ObservationTaskDecorator(observationRegistry));
```

Spring Framework 6.1 introduced a built-in `ContextPropagatingTaskDecorator` that delegates to the `io.micrometer.context.ContextSnapshotFactory` (Micrometer Context Propagation), covering MDC, Reactor Context, and any registered `ThreadLocalAccessor`. Prefer it over a hand-rolled decorator on Spring Boot 3.2+ / Spring Framework 6.1+.

---

## `CompletableFuture` and executor selection

The `xxxAsync` methods on `CompletableFuture` accept an optional `Executor`. Whether you pass one matters:

```java
// ✅ Uses your bounded, instrumented, context-propagating executor
CompletableFuture.supplyAsync(() -> fetchOrder(id), ordersExecutor);

// ❌ Lands on ForkJoinPool.commonPool — shared, unbounded for CPU work, no MDC, no metrics
CompletableFuture.supplyAsync(() -> fetchOrder(id));
```

Consequences of using `commonPool`:

- **No ThreadLocal context** — MDC, Security, Tracing, request scope are not propagated.
- **No Micrometer instrumentation** — Spring Boot's `MeterRegistry` wraps your registered executors but not `commonPool`.
- **`parallelStream` starvation** — `parallelStream` also runs on `commonPool`. Any blocking I/O in `xxxAsync(...)` (or `parallelStream`) eats threads the other one needs.
- **Pool size locked to `Runtime.availableProcessors() - 1`** — too small for IO-heavy work, no way to grow.

Rule: every `xxxAsync` call passes an executor. Same for `Mono.fromCallable(...).subscribeOn(Schedulers.fromExecutor(...))` if you bridge to reactive.

---

## Pattern: structured async with `CompletableFuture.allOf`

```java
CompletableFuture<Customer> customer = CompletableFuture.supplyAsync(
        () -> customerClient.fetch(id), ordersExecutor);
CompletableFuture<List<Item>> items = CompletableFuture.supplyAsync(
        () -> itemClient.fetch(id), ordersExecutor);
CompletableFuture<ShippingQuote> quote = CompletableFuture.supplyAsync(
        () -> shippingClient.quote(id), ordersExecutor);

return CompletableFuture.allOf(customer, items, quote)
        .thenApply(v -> new OrderView(customer.join(), items.join(), quote.join()));
```

Caveats:
- If any of the three fails, the others **keep running** unless you cancel them explicitly. They are not interrupted automatically.
- Cancellation requires `cf.cancel(true)` and the underlying task must respect `Thread.interrupted()`.
- Java 21's `StructuredTaskScope` (preview) / Java 25's structured concurrency solves this cleanly. Prefer it once your baseline is Java 25 — see the `java-performance` skill's `references/virtual-threads.md`.

---

## Migration to virtual threads (Java 21+)

When `spring.threads.virtual.enabled=true` is set:

- **Spring's request-handling executors switch to virtual threads automatically** (Tomcat, Jetty, `@Async`, scheduled tasks).
- **Your custom `ThreadPoolTaskExecutor` beans do not switch automatically.** Decide per workload:
  - **IO-bound `@Async` work** (HTTP calls, DB, file I/O): replace the bean with `Executors.newVirtualThreadPerTaskExecutor()` wrapped in `TaskExecutorAdapter`.
  - **CPU-bound work**: keep the platform-thread `ThreadPoolTaskExecutor` — virtual threads do not help CPU-bound work and the cooperative scheduler can lead to fairness issues.

Virtual-thread executor bean:

```java
@Bean("ordersExecutor")
TaskExecutor ordersExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

- **Don't pool virtual threads.** They are cheap to create; pooling defeats the model.
- **`@Async` pitfalls 1–4 still apply.** Proxy bypass, transaction non-inheritance, exception loss, and ThreadLocal loss are independent of the underlying thread implementation. `TaskDecorator` is still needed.
- On Java 21–23, replace `synchronized { I/O }` blocks with `ReentrantLock` to avoid pinning (fixed in Java 24 via JEP 491).

See the `java-performance` skill's `references/virtual-threads.md` for VT mechanics, pinning detection, and the connection-pool deadlock pattern.

---

## Reactor / WebFlux: context propagation

If the application mixes blocking and reactive code:

```properties
spring.reactor.context-propagation=auto
```

With this property set, ThreadLocal values registered via `ContextRegistry.getInstance().registerThreadLocalAccessor(...)` (MDC, Security, Tracing) are automatically captured into the Reactor `Context` and restored when the chain hops threads.

Without it, blocking interop (`Mono.fromCallable`, `.subscribeOn(Schedulers.boundedElastic())`) loses MDC and tracing across the boundary.

For deep Reactor patterns (operators, schedulers, hot vs cold publishers, backpressure, virtual-time testing), use the `project-reactor` skill.

---

## Logging-as-bottleneck inside `@Async`

```java
log.debug("processing " + order);                        // ❌ toString runs even if level=INFO
log.debug("processing {}", order);                       // ✅ skipped if level=INFO
log.atDebug().log(() -> "processing " + jsonify(order)); // ✅ functional — heavy formatter only runs if level=DEBUG
```

`order.toString()` on a JPA entity can trigger lazy-load (network call), allocate, and serialise — silently blowing up CPU and DB time inside what should be an async background task.

---

## Sentry / observability integration

For applications using Sentry, set the environment explicitly so cross-environment filtering works in the UI:

```properties
sentry.environment=production
```

This belongs in `application-{profile}.properties` (e.g. `application-prod.properties`), not in the base file.

Async stack traces are notoriously hard to debug. Two things help:

1. **`TaskDecorator` propagates the Sentry hub / scope** — so a request-tagged Sentry scope is attached to the async work. The built-in `ContextPropagatingTaskDecorator` handles this via the Micrometer context bridge.
2. **Always return `CompletableFuture<T>` from `@Async`** when you need observability — exceptions in `void`-returning `@Async` methods bypass Spring's `@RestControllerAdvice` and require the `AsyncUncaughtExceptionHandler` to forward to Sentry manually.

---

## Common mistakes — quick reference

| Mistake | Symptom | Fix |
|---|---|---|
| `@Async` self-call | Method runs synchronously despite annotation | Inject `self` via `@Lazy` or move to separate bean |
| `@Async` on `void` with no handler | Exceptions silently dropped | Register `AsyncUncaughtExceptionHandler`; or return `CompletableFuture` |
| `CompletableFuture.supplyAsync(task)` (no executor) | MDC/Security lost; metrics missing; `parallelStream` starves | Always pass a Spring-managed `Executor` |
| No `TaskDecorator` on `ThreadPoolTaskExecutor` | `MDC` shows no `traceId` in async log lines; `SecurityContextHolder.getContext().getAuthentication()` is null | Add `ContextPropagatingTaskDecorator` (Spring 6.1+ built-in) |
| Unbounded queue (`Executors.newFixedThreadPool`) | Memory grows under load; tasks queue indefinitely | Bounded `queueCapacity` + `CallerRunsPolicy` |
| Mixing CPU + IO on one executor | One workload starves the other | Separate `ThreadPoolTaskExecutor` per workload |
| `synchronized { I/O }` on virtual threads (Java 21–23) | VT pinning; carrier-thread exhaustion under load | `ReentrantLock` (or upgrade to Java 24+) |
| `@Async` method that does `dataSource.getConnection()` (or JPA call) without `@Transactional` | Connection acquired, never released cleanly | Add `@Transactional` on the `@Async` method |
| `@Async` fanning out N parallel DB calls inside an outer `@Transactional` | `hikaricp_connections_acquire_seconds` spikes; pool starvation | Size pool for N+1; or use `@TransactionalEventListener(AFTER_COMMIT)` to defer |

---

## See also

- `SKILL.md` § *Asynchronous Execution (@Async, ThreadPoolTaskExecutor)* — the brief inline summary
- `SKILL.md` § *Virtual Threads* — when to enable, JDK version notes
- `project-reactor` skill — for `WebClient`, `Mono`/`Flux` operator selection, Reactor Context, BlockHound
- `hibernate-jpa-validator` skill, `references/spring-transactions.md` — for `@Transactional` propagation, `@TransactionalEventListener`, `LazyConnectionDataSourceProxy`, and the connection-acquisition lifecycle
- `java-performance` skill, `references/concurrency-and-threads.md` — for Goetz pool-sizing formula, lock profiling, race-bug heuristics
- `java-performance` skill, `references/virtual-threads.md` — for VT pinning, structured concurrency, ScopedValues, VT vs reactive
