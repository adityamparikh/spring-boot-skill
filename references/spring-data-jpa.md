# Spring Data JPA

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

## Transactions

- Annotate write services with `@Transactional`.
- Read-only methods get `@Transactional(readOnly = true)` for connection-pool and Hibernate flush optimizations.

## Hibernate Version Notes

- **3.5.x**: Hibernate ORM 6.x. `GenerationType.SEQUENCE` defaults are sane; `IDENTITY` blocks JDBC batching.
- **4.0.x**: Hibernate ORM 7.1.x. Some entity-mapping defaults tighten -- review `references/spring-framework7.md` in the `spring-boot-4-migration` skill before upgrading.

For deep JPA review (N+1 detection, fetch plans, batch tuning, connection pooling, query count assertions), use the `hibernate-jpa-validator` skill -- it activates on JPA-specific code review and will not duplicate guidance here.
