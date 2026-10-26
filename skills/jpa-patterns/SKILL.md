---
name: jpa-patterns
description: JPA/Hibernate patterns and common pitfalls (N+1, lazy loading, transactions, queries) for Quarkus/Panache. Use when user has JPA performance issues, LazyInitializationException, or asks about entity relationships and fetching strategies.
---

# JPA Patterns Skill

Best practices and common pitfalls for JPA/Hibernate in Quarkus applications (Hibernate ORM with Panache).

## When to Use
- User mentions "N+1 problem" / "too many queries"
- LazyInitializationException errors
- Questions about fetch strategies (EAGER vs LAZY)
- Transaction management issues
- Entity relationship design
- Query optimization

---

## Quick Reference: Common Problems

| Problem | Symptom | Solution |
|---------|---------|----------|
| N+1 queries | Many SELECT statements | JOIN FETCH, entity graphs |
| LazyInitializationException | Error outside transaction | JOIN FETCH, DTO projection, keep access inside `@Transactional` |
| Slow queries | Performance issues | Pagination, projections, indexes |
| Dirty checking overhead | Slow updates | Avoid unnecessary `@Transactional`, use DTOs for pure reads |
| Lost updates | Concurrent modifications | Optimistic locking (@Version) |

---

## N+1 Problem

> The #1 JPA performance killer

### The Problem

```java
// ❌ BAD: N+1 queries
@Entity
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

// This innocent code...
List<Author> authors = authorRepository.listAll();  // 1 query
for (Author author : authors) {
    System.out.println(author.getBooks().size());   // N queries!
}
// Result: 1 + N queries (if 100 authors = 101 queries)
```

### Solution 1: JOIN FETCH (JPQL)

```java
// ✅ GOOD: Single query with JOIN FETCH
@ApplicationScoped
public class AuthorRepository implements PanacheRepository<Author> {

    public List<Author> findAllWithBooks() {
        return getEntityManager()
            .createQuery("SELECT DISTINCT a FROM Author a JOIN FETCH a.books", Author.class)
            .getResultList();
    }
}

// Usage - single query
List<Author> authors = authorRepository.findAllWithBooks();
```

### Solution 2: Named Entity Graph

```java
// ✅ GOOD: Entity graph for declarative fetching (pure JPA/Hibernate feature)
@Entity
@NamedEntityGraph(
    name = "Author.withBooks",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author {
    // ...
}

@ApplicationScoped
public class AuthorRepository implements PanacheRepository<Author> {

    public List<Author> findAllWithBooksViaGraph() {
        EntityGraph<?> graph = getEntityManager().getEntityGraph("Author.withBooks");
        return getEntityManager()
            .createQuery("SELECT a FROM Author a", Author.class)
            .setHint("jakarta.persistence.loadgraph", graph)
            .getResultList();
    }
}
```

### Solution 3: Batch Fetching

```java
// ✅ GOOD: Batch fetching (Hibernate-specific)
@Entity
public class Author {

    @OneToMany(mappedBy = "author")
    @BatchSize(size = 25)  // Fetch 25 at a time
    private List<Book> books;
}
```

```properties
# Or globally in application.properties
quarkus.hibernate-orm.fetch.batch-size=25
```

### Detecting N+1

```properties
# application.properties - enable SQL logging to detect N+1
# (build-time property - requires a rebuild to toggle)
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.log.format-sql=true
quarkus.hibernate-orm.log.bind-parameters=true

# Runtime alternative - toggle without rebuilding
quarkus.log.category."org.hibernate.SQL".level=DEBUG
```

---

## Lazy Loading

### FetchType Basics

```java
@Entity
public class Order {

    // LAZY: Load only when accessed (default for collections)
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;

    // EAGER: Always load immediately (default for @ManyToOne, @OneToOne)
    @ManyToOne(fetch = FetchType.EAGER)  // ⚠️ Usually bad
    private Customer customer;
}
```

### Best Practice: Default to LAZY

```java
// ✅ GOOD: Always use LAZY, fetch when needed
@Entity
public class Order {

    @ManyToOne(fetch = FetchType.LAZY)  // Override EAGER default
    private Customer customer;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

### LazyInitializationException

```java
// ❌ BAD: Accessing lazy field outside transaction
@ApplicationScoped
public class OrderService {

    public Order getOrder(Long id) {
        return orderRepository.findById(id);
    }
}

// In a REST resource (no transaction)
Order order = orderService.getOrder(1L);
order.getItems().size();  // 💥 LazyInitializationException!
```

### Solutions for LazyInitializationException

**Solution 1: JOIN FETCH in query**
```java
// ✅ Fetch needed associations in query
public Optional<Order> findByIdWithItems(Long id) {
    return getEntityManager()
        .createQuery("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id", Order.class)
        .setParameter("id", id)
        .getResultStream()
        .findFirst();
}
```

**Solution 2: @Transactional on service method**
```java
// ✅ Keep transaction open while accessing
@ApplicationScoped
public class OrderService {

    @Transactional
    public OrderDTO getOrderWithItems(Long id) {
        Order order = orderRepository.findByIdOptional(id).orElseThrow();
        // Access within transaction - lazy collection loads here
        int itemCount = order.getItems().size();
        return new OrderDTO(order, itemCount);
    }
}
```

> Quarkus's `@Transactional` (`jakarta.transaction.Transactional`) has no `readOnly` attribute and no built-in read-only optimization hint.

**Solution 3: DTO projection (best for read-only)**
```java
// ✅ Avoid lazy loading entirely - query only what you need
public Optional<OrderSummary> findSummaryById(Long id) {
    return getEntityManager()
        .createQuery("SELECT new com.example.dto.OrderSummary(o.id, o.status, SIZE(o.items)) " +
                     "FROM Order o WHERE o.id = :id", OrderSummary.class)
        .setParameter("id", id)
        .getResultStream()
        .findFirst();
}
```

---

## Transaction Management

### Read-Only Optimization

```java
@ApplicationScoped
public class ProductService {
    private final ProductRepository repo;

    public ProductService(ProductRepository repo) {
        this.repo = repo;
    }

    public List<Product> findAll() {
        return repo.listAll();  // Plain read: no @Transactional needed for a single operation
    }

    @Transactional  // Required for multi-step writes
    public Product create(ProductRequest request) {
        var product = new Product();
        product.setName(request.name());
        repo.persist(product);
        return product;
    }
}
```

A single Panache call outside an active transaction runs in its own implicit transaction automatically — you only need `@Transactional` explicitly for multi-step writes, or to keep a lazy association reachable across several calls.

### Self-Invocation and Interceptors

Quarkus supports intercepted self-invocation: a call through `this` to a non-private method
annotated with `@Transactional` (or any interceptor binding) is intercepted, so a transaction
starts. This is a Quarkus (ArC) feature; the CDI specification leaves it undefined, so code
shared with another CDI runtime should not rely on it.

```java
@ApplicationScoped
public class OrderService {

    public void processOrders(List<Long> ids) {
        for (Long id : ids) {
            processOrder(id);  // intercepted in Quarkus: each call runs in its own transaction
        }
    }

    @Transactional
    public void processOrder(Long id) {
        // ...
    }
}
```

Pitfalls that remain:
- An interceptor binding on a `private` method fails the build by default
  (`quarkus.arc.fail-on-intercepted-private-method=true`); make the method package-private or public.
- Each self-invoked `@Transactional` call uses the default `REQUIRED` semantics: if the caller is
  already transactional, the calls join its transaction instead of committing one by one. Use
  `@Transactional(REQUIRES_NEW)` when each item must commit independently.
- Extracting the transactional method into a separate bean is still a valid design choice when it
  clarifies responsibilities, but it is not required for interception.

See the [CDI reference: intercepted self-invocation](https://quarkus.io/guides/cdi-reference).

---

## Optimistic Locking

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private BigDecimal price;

    @Version
    private Long version;  // Hibernate checks this on update

    // getters/setters
}

// Handling optimistic lock failure
@ApplicationScoped
public class ProductService {
    private final ProductRepository repo;

    public ProductService(ProductRepository repo) {
        this.repo = repo;
    }

    @Transactional
    public Product updatePrice(Long id, BigDecimal newPrice) {
        Product product = repo.findByIdOptional(id)
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));
        product.setPrice(newPrice);
        try {
            repo.flush();  // Force the version check now instead of at commit
            return product;
        } catch (OptimisticLockException e) {
            throw new ClientErrorException("Product was modified by another user", Response.Status.CONFLICT);
        }
    }
}
```

---

## Modern Hibernate Features

### Jakarta Persistence

```java
// Always use jakarta.persistence.* (not javax.persistence.*)
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
```

### UUID Generation

```java
@Entity
public class Event {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)  // Native UUID generation
    private UUID id;

    private String name;

    // getters/setters
}
```

### Soft Delete

```java
// Built-in soft delete support (Hibernate ORM 6.4+)
@Entity
@SoftDelete  // Adds 'deleted' boolean column, filters automatically
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // getters/setters
}

// Custom column name
@Entity
@SoftDelete(columnName = "is_active", converter = NumericBooleanConverter.class)
public class Product {
    // ...
}
```

### Record as DTO in Queries

```java
// Records work natively in JPQL constructor expressions
public record ProductSummary(Long id, String name, BigDecimal price) {}

@ApplicationScoped
public class ProductRepository implements PanacheRepository<Product> {

    public List<ProductSummary> findExpensiveProducts(BigDecimal minPrice) {
        return getEntityManager()
            .createQuery("SELECT new com.example.dto.ProductSummary(p.id, p.name, p.price) " +
                         "FROM Product p WHERE p.price > :minPrice", ProductSummary.class)
            .setParameter("minPrice", minPrice)
            .getResultList();
    }
}
```

---

## Query Best Practices

### Pagination

```java
// ✅ Always paginate unbounded queries
@ApplicationScoped
public class UserRepository implements PanacheRepository<User> {

    public List<User> findActive(int pageIndex, int pageSize) {
        return find("active", Sort.by("createdAt").descending(), true)
            .page(Page.of(pageIndex, pageSize))
            .list();
    }
}

// Usage
List<User> users = userRepository.findActive(0, 20);
long totalCount = userRepository.count("active", true);
```

### DTO / Record Projections

```java
// ✅ Fetch only the columns you need via a JPQL constructor expression
public record UserNameOnly(String firstName, String lastName) {}

@ApplicationScoped
public class UserRepository implements PanacheRepository<User> {

    public List<UserNameOnly> findAllActiveNames() {
        return getEntityManager()
            .createQuery("SELECT new com.example.dto.UserNameOnly(u.firstName, u.lastName) " +
                         "FROM User u WHERE u.active = true", UserNameOnly.class)
            .getResultList();
    }
}
```

---

## Quick Decision Guide

| Situation | Solution |
|-----------|----------|
| N+1 in loop | JOIN FETCH or named entity graph |
| LazyInitializationException | JOIN FETCH in query or `@Transactional` on service |
| Read-only queries slow | Avoid `@Transactional` on single reads; use DTO projections |
| Concurrent updates | `@Version` + optimistic locking |
| Unbounded result set | Pagination with Panache `Page.of()` |
| Need only few columns | DTO/record projection (JPQL constructor expression) |
| Soft delete required | `@SoftDelete` |
| UUID primary key | `@GeneratedValue(strategy = GenerationType.UUID)` |
