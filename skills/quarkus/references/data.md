# Data Access - Hibernate ORM with Panache

## Panache Entity Pattern (Active Record)

```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_email", columnList = "email", unique = true),
    @Index(name = "idx_username", columnList = "username")
})
public class User extends PanacheEntity {

    @Column(nullable = false, unique = true, length = 100)
    public String email;

    @Column(nullable = false, length = 100)
    public String password;

    @Column(nullable = false, unique = true, length = 50)
    public String username;

    @Column(nullable = false)
    public Boolean active = true;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    public List<Address> addresses = new ArrayList<>();

    @ManyToMany
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    public Set<Role> roles = new HashSet<>();

    @Column(nullable = false, updatable = false)
    public LocalDateTime createdAt;

    @Column(nullable = false)
    public LocalDateTime updatedAt;

    @Version
    public Long version;

    @PrePersist
    void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = createdAt;
    }

    @PreUpdate
    void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    // Helper methods for bidirectional relationships
    public void addAddress(Address address) {
        addresses.add(address);
        address.user = this;
    }

    public void removeAddress(Address address) {
        addresses.remove(address);
        address.user = null;
    }
}
```

`PanacheEntity` (active record) gives the entity a generated `Long id` and public-field style with no getters/setters to write — that's Panache's own boilerplate removal. Timestamps use the standard JPA `@PrePersist`/`@PreUpdate` lifecycle callbacks; there's no `@CreatedDate`/`@LastModifiedDate`/auditing-listener combo bundled by default. For `createdBy`/`updatedBy`-style user auditing, set the fields explicitly in the service layer (inject `SecurityIdentity`, see `security.md`) rather than reaching for an entity listener — JPA listener classes aren't CDI beans by default, so DI inside one needs an extra workaround that's rarely worth it.

### Shared Audit Fields in a `@MappedSuperclass`

When several entities carry the same timestamps, declare them and their callbacks once:

```java
@MappedSuperclass
public abstract class AuditableEntity extends PanacheEntity {

    @Column(nullable = false, updatable = false)
    public LocalDateTime createdAt;

    @Column(nullable = false)
    public LocalDateTime updatedAt;

    @PrePersist
    void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = createdAt;
    }

    @PreUpdate
    void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}

@Entity
public class Order extends AuditableEntity {
    // inherits id, createdAt, updatedAt and the lifecycle callbacks
}
```

With the repository pattern, the superclass doesn't need to extend `PanacheEntity`: declare the `@Id` in it (or in each entity) and use `PanacheRepository<Order>` as usual. Hibernate's `@CreationTimestamp`/`@UpdateTimestamp` are a no-callback alternative for the two timestamps.

## Panache Repository Pattern

Prefer this over the active-record style when the service layer should own persistence logic explicitly:

```java
@ApplicationScoped
public class UserRepository implements PanacheRepository<User> {

    public Optional<User> findByEmail(String email) {
        return find("email", email).firstResultOptional();
    }

    public Optional<User> findByUsername(String username) {
        return find("username", username).firstResultOptional();
    }

    public boolean existsByEmail(String email) {
        return count("email", email) > 0;
    }

    public Optional<User> findByEmailWithRoles(String email) {
        return getEntityManager()
            .createQuery("SELECT u FROM User u LEFT JOIN FETCH u.roles WHERE u.email = :email", User.class)
            .setParameter("email", email)
            .getResultStream()
            .findFirst();
    }

    public List<User> findActiveUsersSince(LocalDateTime since) {
        return find("active = true and createdAt >= ?1", since).list();
    }

    @Transactional
    public int deactivateInactiveUsers(LocalDateTime threshold) {
        return update("active = false where lastLoginAt < ?1", threshold);
    }

    // Projection for read-only DTOs
    public List<UserSummary> findAllActiveSummaries() {
        return getEntityManager()
            .createQuery("SELECT new com.example.dto.UserSummary(u.id, u.username, u.email) " +
                         "FROM User u WHERE u.active = true", UserSummary.class)
            .getResultList();
    }
}
```

`find()`/`count()`/`update()`/`delete()` take a Panache query (plain field names and a WHERE clause, or a full HQL-style query) with either named (`:email`) or positional (`?1`) parameters — always parameterized, never string-concatenated. `update()` runs a bulk UPDATE statement and must run inside a transaction.

## Dynamic Queries

Panache has no dedicated composable-predicate API — it favors building the query string and a parameter map explicitly instead:

```java
@ApplicationScoped
public class UserRepository implements PanacheRepository<User> {

    public List<User> search(UserSearchCriteria criteria) {
        var conditions = new ArrayList<String>();
        var params = new HashMap<String, Object>();

        conditions.add("active = true");
        if (criteria.email() != null) {
            conditions.add("email = :email");
            params.put("email", criteria.email());
        }
        if (criteria.createdAfter() != null) {
            conditions.add("createdAt >= :createdAfter");
            params.put("createdAfter", criteria.createdAfter());
        }

        String query = String.join(" and ", conditions);
        return find(query, params).list();
    }
}
```

For genuinely complex, composable filtering, the standard JPA `CriteriaBuilder` API works unchanged in Quarkus; the Quarkiverse `quarkus-blaze-persistence` extension is worth reaching for if the project needs a fluent, composable query builder on top of that.

## Transaction Management

```java
@ApplicationScoped
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository orderRepository;
    private final OrderEventRepository orderEventRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final NotificationService notificationService;

    public OrderService(OrderRepository orderRepository, OrderEventRepository orderEventRepository,
                        PaymentService paymentService, InventoryService inventoryService,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.orderEventRepository = orderEventRepository;
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
        this.notificationService = notificationService;
    }

    @Transactional
    public Order createOrder(OrderCreateRequest request) {
        // All operations in a single transaction
        Order order = new Order();
        order.customerId = request.customerId();
        order.status = OrderStatus.PENDING;

        request.items().forEach(item -> {
            inventoryService.reserveStock(item.productId(), item.quantity());
            order.addItem(item);
        });

        orderRepository.persist(order);

        try {
            paymentService.processPayment(order);
            order.status = OrderStatus.PAID;
        } catch (PaymentException e) {
            order.status = OrderStatus.PAYMENT_FAILED;
            throw e; // Transaction will roll back
        }

        return order;
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void logOrderEvent(Long orderId, String event) {
        // Separate transaction - commits even if the parent rolls back
        OrderEvent orderEvent = new OrderEvent(orderId, event);
        orderEventRepository.persist(orderEvent);
    }

    @Transactional(dontRollbackOn = NotificationException.class)
    public void completeOrder(Long orderId) {
        Order order = orderRepository.findByIdOptional(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        order.status = OrderStatus.COMPLETED;

        // Won't roll back the transaction if the notification fails
        try {
            notificationService.sendCompletionEmail(order);
        } catch (NotificationException e) {
            log.error("Failed to send notification for order {}", orderId, e);
        }
    }
}
```

`jakarta.transaction.Transactional` is the JTA standard annotation — it has no `readOnly` attribute (skip the annotation entirely on plain reads instead), but it does carry propagation (`Transactional.TxType.REQUIRES_NEW`, `MANDATORY`, `SUPPORTS`, `NOT_SUPPORTED`, `NEVER`) and rollback control (`rollbackOn`, `dontRollbackOn`) — standard JTA, not a Quarkus extra.

A Panache entity loaded inside a transaction is dirty-checked and flushed automatically at commit — there's no `repository.save(entity)` call needed for an already-managed entity (see `order.status = ...` above); `persist()` is only for brand-new entities.

## Projections

Panache has no interface-based projection feature (a getter interface auto-backed by a dynamic proxy). The portable, JPA-standard approach is a record/DTO built with a JPQL constructor expression:

```java
public record UserSummary(Long id, String username, String email) {}

@ApplicationScoped
public class UserRepository implements PanacheRepository<User> {

    public List<UserSummary> findAllActiveSummaries() {
        return getEntityManager()
            .createQuery("SELECT new com.example.dto.UserSummary(u.id, u.username, u.email) " +
                         "FROM User u WHERE u.active = true", UserSummary.class)
            .getResultList();
    }
}
```

See the standalone `jpa-patterns` skill for the full set of N+1/lazy-loading/optimistic-locking patterns this projection technique is part of.

## Query Optimization

```java
// Entity-level optimization: batch fetching on collections (Hibernate-specific, unchanged from plain JPA)
@Entity
public class User extends PanacheEntity {
    // ...
    @BatchSize(size = 25)
    @OneToMany(mappedBy = "user")
    public List<Order> orders;
}

// Repository-level optimization: JOIN FETCH, pagination, native queries
@ApplicationScoped
public class UserRepository implements PanacheRepository<User> {

    public List<User> findAllActiveWithAssociations() {
        return getEntityManager()
            .createQuery("SELECT DISTINCT u FROM User u " +
                         "LEFT JOIN FETCH u.addresses " +
                         "LEFT JOIN FETCH u.roles " +
                         "WHERE u.active = true", User.class)
            .getResultList();
    }

    public List<User> findActive(int pageIndex, int pageSize) {
        return find("active", true).page(Page.of(pageIndex, pageSize)).list();
    }

    @SuppressWarnings("unchecked")
    public List<User> findFrequentBuyers(LocalDateTime since, int minOrders) {
        return getEntityManager().createNativeQuery("""
            SELECT u.* FROM users u
            INNER JOIN orders o ON u.id = o.user_id
            WHERE o.created_at >= :since
            GROUP BY u.id
            HAVING COUNT(o.id) >= :minOrders
            """, User.class)
            .setParameter("since", since)
            .setParameter("minOrders", minOrders)
            .getResultList();
    }
}
```

## Database Migrations (Flyway or Liquibase)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-flyway</artifactId>
</dependency>
```

```properties
quarkus.flyway.migrate-at-start=true
quarkus.hibernate-orm.database.generation=validate
```

```sql
-- src/main/resources/db/migration/V1__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(100) NOT NULL,
    username VARCHAR(50) NOT NULL UNIQUE,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_active ON users(active);
```

This is Flyway's own `V<version>__description.sql` naming convention. With `quarkus-liquibase` the equivalent is `quarkus.liquibase.migrate-at-start=true` and changelog files under `src/main/resources/db/changeLog/`. Either extension, combined with the `quarkus-kubernetes` extension, gets its migration automatically turned into a Kubernetes init `Job` that gates the Deployment's rollout — see the `kubernetes-specialist` agent.

## Quick Reference

| Annotation / API | Purpose |
|------------------|---------|
| `@Entity` / `extends PanacheEntity` | JPA entity, active-record style with Panache |
| `implements PanacheRepository<T>` | Repository pattern, explicit persistence logic |
| `@Table` | Table details and indexes |
| `@Id` / `@GeneratedValue` | Primary key field and generation strategy (only needed outside `PanacheEntity`) |
| `@Column` | Column constraints and mapping |
| `@OneToMany`/`@ManyToOne`/`@ManyToMany` | Entity relationships (unchanged JPA) |
| `@Transactional` | Declares transaction boundaries (`jakarta.transaction.Transactional`) |
| `find()`/`list()`/`page()` | Panache query DSL |
| `getEntityManager()` | Drop down to plain JPQL/native SQL for anything Panache's DSL can't express |
| `@Version` | Optimistic locking version field |
