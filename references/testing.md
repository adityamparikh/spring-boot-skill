# Testing Spring Components

Match the test slice to what you are testing. Full-context tests are slow -- use them only when integration is the point.

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

## Mock Annotations by Version

- **3.5.x**: prefer `@MockitoBean` / `@MockitoSpyBean` (introduced in 3.4). `@MockBean` / `@SpyBean` still work but are deprecated.
- **4.0.x**: `@MockBean` and `@SpyBean` are **removed**. Use `@MockitoBean` / `@MockitoSpyBean`.

## HTTP Test Clients

- 3.5.x: `MockMvc`, `MockMvcTester` (3.4+), or `WebTestClient` (WebFlux).
- 4.0.x: same plus `RestTestClient` for `RestClient`-style assertions in `@SpringBootTest`.

## JUnit Version

- 3.5.x: JUnit 5 (Jupiter).
- 4.0.x: JUnit 6.

## Testcontainers

Use `@Testcontainers` + `@ServiceConnection` (Boot 3.1+) so Spring Boot wires the container's connection details directly into the test context -- no manual `@DynamicPropertySource` needed.

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
