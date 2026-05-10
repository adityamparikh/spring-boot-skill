# REST Controllers

Keep controllers thin: bind input, delegate to a service, return a response. Validation, persistence, and orchestration belong outside the controller body.

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

## Records for DTOs

Use records for request and response DTOs. They are immutable, support Bean Validation annotations on parameters, and Jackson serializes them with no extra config.

```java
record CreateOrderRequest(@NotBlank String sku,
                          @Positive int quantity,
                          @Email String customerEmail) {}
```

## API Versioning

- **3.5.x**: header (`@RequestMapping(headers = "X-API-Version=2")`) or path prefix (`/api/v2/...`).
- **4.0.x**: prefer Spring Framework 7's built-in `version` attribute:

  ```java
  @GetMapping(path = "/{id}", version = "2")
  Order getV2(@PathVariable UUID id) { ... }
  ```

For a full versioning strategy comparison (path / header / media-type / query), see the `spring-boot-4-migration` skill's `references/api-versioning.md`.
