---
name: quarkus
description: Quarkus development - REST APIs, Panache, Security, Testing, and Cloud-native patterns. Use for building enterprise Java applications with Quarkus.
metadata:
  version: "0.1.0"
  domain: backend
  triggers: Quarkus, Panache, Hibernate ORM, Jakarta REST, CDI, MicroProfile, Java REST API, Microservices Java
  role: specialist
  scope: implementation
  output-format: code
---

# Quarkus Skill

Enterprise Quarkus development with focus on clean architecture and production-ready, cloud-native (and optionally native-image-compilable) code.

## Core Workflow

1. **Analyze** - Understand requirements, identify service boundaries, APIs, data models
2. **Design** - Plan architecture, confirm design before coding
3. **Implement** - Build with constructor injection and layered architecture
4. **Secure** - Add Quarkus Security (OIDC/JWT), method security; verify tests pass
5. **Test** - Write unit tests and `@QuarkusTest` integration tests; run `./mvnw test` and confirm all pass
6. **Deploy** - Health checks come from SmallRye Health automatically; validate `/q/health` returns UP

## Quick Start Templates

### Entity (Panache)
```java
@Entity
@Table(name = "products")
public class Product extends PanacheEntity {

    @NotBlank
    public String name;

    @DecimalMin("0.0")
    public BigDecimal price;

    // Panache active-record entities use public fields instead of getters/setters -
    // that's Panache's own boilerplate removal, not a "skip validation" shortcut.
}
```

### Repository
```java
@ApplicationScoped
public class ProductRepository implements PanacheRepository<Product> {

    public List<Product> findByNameContainingIgnoreCase(String name) {
        return find("lower(name) like ?1", "%" + name.toLowerCase() + "%").list();
    }
}
```

### Service
```java
@ApplicationScoped
public class ProductService {
    private final ProductRepository repo;

    public ProductService(ProductRepository repo) {
        this.repo = repo;
    }

    public List<ProductResponse> search(String name) {
        return repo.findByNameContainingIgnoreCase(name).stream()
            .map(ProductResponse::from)
            .toList();
    }

    @Transactional
    public ProductResponse create(ProductRequest request) {
        var product = new Product();
        product.name = request.name();
        product.price = request.price();
        repo.persist(product);
        return ProductResponse.from(product);
    }
}
```

### REST Resource
```java
@Path("/api/v1/products")
public class ProductResource {
    private final ProductService service;

    public ProductResource(ProductService service) {
        this.service = service;
    }

    @GET
    public List<ProductResponse> search(@QueryParam("name") @DefaultValue("") String name) {
        return service.search(name);
    }

    @POST
    @Status(201)
    public ProductResponse create(@Valid ProductRequest request) {
        return service.create(request);
    }
}
```

### DTOs (Records)
```java
public record ProductRequest(
    @NotBlank String name,
    @DecimalMin("0.0") BigDecimal price
) {}

public record ProductResponse(Long id, String name, BigDecimal price) {
    public static ProductResponse from(Product product) {
        return new ProductResponse(product.id, product.name, product.price);
    }
}
```

### Global Exception Handler
```java
@Provider
public class GlobalExceptionMapper implements ExceptionMapper<Exception> {

    @Override
    public Response toResponse(Exception ex) {
        return switch (ex) {
            case ConstraintViolationException e -> Response.status(400)
                .entity(Map.of("errors", e.getConstraintViolations().stream()
                    .collect(Collectors.toMap(
                        v -> v.getPropertyPath().toString(),
                        v -> v.getMessage() != null ? v.getMessage() : "Invalid"))))
                .build();
            case EntityNotFoundException e -> Response.status(404)
                .entity(Map.of("error", e.getMessage()))
                .build();
            default -> Response.status(500)
                .entity(Map.of("error", "Unexpected error"))
                .build();
        };
    }
}
```

### Test Slice
```java
@QuarkusTest
class ProductResourceTest {

    @InjectMock
    ProductService service;

    @Test
    void createProduct_validRequest_returns201() {
        var response = new ProductResponse(1L, "Widget", new BigDecimal("10.00"));
        when(service.create(any())).thenReturn(response);

        given()
            .contentType(ContentType.JSON)
            .body("""
                {"name":"Widget","price":10.0}
                """)
            .when().post("/api/v1/products")
            .then()
                .statusCode(201)
                .body("name", is("Widget"));
    }
}
```

## Reference Guide

Load detailed patterns based on context:

| Topic | Reference | When to Load |
|-------|-----------|-------------|
| Web/REST | `references/web.md` | JAX-RS resources, validation, exception handling, REST clients |
| Data Access | `references/data.md` | Panache, repositories, transactions, queries |
| Security | `references/security.md` | Quarkus Security, OIDC, JWT, method security, Keycloak Dev Services |
| Cloud/Config | `references/cloud.md` | Service discovery (Stork), resilience, virtual threads, HTTP clients, native image, OTel tracing |
| Testing | `references/testing.md` | Unit, integration tests, RestAssured, Dev Services |

For deeper, standalone dives also available as separate skills in this plugin: `jpa-patterns`, `testing-patterns`, `java-modern`, `design-patterns`, `code-quality`, `logging-patterns`.

## Constraints

### MUST DO
- Constructor injection, with a single constructor (CDI/ArC picks it up automatically — `@Inject` on it is never needed)
- `@Valid` on request parameters/bodies
- `@Transactional` for multi-step writes
- Skip `@Transactional` on simple single reads — Quarkus's `@Transactional` has no `readOnly` attribute
- Type-safe config with `@ConfigMapping`
- Global exception handling with a `@Provider`-annotated `ExceptionMapper`
- Externalize secrets (environment variables or mounted files, not properties files)

### MUST NOT DO
- Field injection (`@Inject` on fields)
- Skip input validation on endpoints
- Mix blocking (`@RunOnVirtualThread`/plain) and Mutiny-reactive return types in the same resource without a clear reason
- Store secrets in `application.properties`
- Hardcode URLs, credentials, environment values, or a group ID/package convention the project doesn't already use

## Architecture Patterns

**Project Structure:**
```
src/main/java/<base-package>/
├── resource/    # JAX-RS endpoints
├── service/     # Business logic
├── repository/  # Data access (Panache)
├── entity/      # Panache/JPA entities
├── dto/         # Request/Response DTOs
├── config/      # @ConfigMapping interfaces
└── exception/   # Custom exceptions + ExceptionMapper
```

**Layering:**
- Resource → Service → Repository
- Resource handles HTTP, validation
- Service handles business logic, transactions
- Repository handles data persistence

**Clean Architecture Principles:**
- Domain models independent of frameworks
- Use case driven design
- Dependency inversion (interfaces)
- Clear boundaries between layers

## Common Annotations

| Annotation | Purpose |
|------------|---------|
| `@Path` / `@GET`/`@POST`/... | JAX-RS resource and HTTP method mapping |
| `@ApplicationScoped` | CDI singleton-scoped bean (services, repositories) |
| `@Transactional` | Transaction management (`jakarta.transaction.Transactional`) |
| `@Valid` | Trigger Bean Validation |
| `@ConfigMapping` | Bind a group of properties to a type-safe interface |
| `@RolesAllowed` | Method/class-level role-based access control |

## CDI Scopes and Pitfalls

| Scope | Client proxy? | Notes |
|-------|---------------|-------|
| `@ApplicationScoped` | Yes (lazy) | Default for services and repositories; mockable with `@InjectMock` |
| `@Singleton` | No | Slightly cheaper; without a proxy the instance is created as soon as it is injected; not mockable with `@InjectMock` by default |
| `@RequestScoped` | Yes | Safe to inject into an `@ApplicationScoped` bean — the proxy resolves the current request's instance |
| `@Dependent` | No | New instance per injection point, bound to the owner's lifecycle |

- Creating a bean with `new` bypasses CDI: no injection and no interceptors (`@Transactional`, `@Timed`, ...). Fine in a plain unit test, wrong in production code.
- `@ApplicationScoped`/`@Singleton` beans are shared across threads — keep them stateless, or make shared state thread-safe.
- Don't copy a request-scoped value (e.g. `identity.getPrincipal()`) into a field of an application-scoped bean; inject the proxy and read it per call.
- Several implementations of one type, or lazy lookup: inject `@Any Instance<T>`.
- Quarkus removes unused beans at build time; a bean looked up only dynamically (e.g. `CDI.current().select(...)`) needs `@Unremovable`.
- Startup work: observe `StartupEvent` (`void onStart(@Observes StartupEvent e)`) or annotate the bean with `@Startup`.

## Reactive Resource (Mutiny)

```java
@Path("/api/v1/orders")
public class OrderResource {
    private final OrderService orderService;

    public OrderResource(OrderService orderService) {
        this.orderService = orderService;
    }

    @GET
    @Path("/{id}")
    public Uni<Response> getOrder(@PathParam("id") UUID id) {
        return orderService.findById(id)
            .map(order -> Response.ok(order).build())
            .onItem().ifNull().continueWith(Response.status(404).build());
    }

    @POST
    public Uni<OrderDto> createOrder(@Valid CreateOrderRequest request) {
        return orderService.create(request);
    }
}
```

RestAssured tests this resource exactly like a blocking one — no separate reactive test client is needed, since it just drives real HTTP calls regardless of what runs behind the endpoint (see `references/testing.md`).

## Quarkus Security — JWT

```properties
# application.properties - validate bearer tokens, no custom filter needed
mp.jwt.verify.publickey.location=publicKey.pem
mp.jwt.verify.issuer=https://issuer.example.com
quarkus.http.auth.permission.public.paths=/q/health/*,/api/v1/auth/*
quarkus.http.auth.permission.public.policy=permit
```

```java
@Path("/api/v1/orders")
public class OrderResource {

    @GET
    @RolesAllowed("USER")
    public List<OrderDto> listMine(@Context SecurityContext ctx) {
        String username = ctx.getUserPrincipal().getName();
        // ...
    }

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public void delete(@PathParam("id") Long id) { /* ... */ }
}
```

`quarkus-smallrye-jwt` validates the `Authorization: Bearer` header declaratively from the extension/config above — there is no Java security configuration class to write (see `references/security.md`).

## Knowledge Base

Quarkus, Java 21+, Jakarta REST (Quarkus REST), Hibernate ORM with Panache, Quarkus Security, OIDC/JWT (SmallRye JWT), Hibernate, Mutiny, SmallRye Reactive Messaging, SmallRye Fault Tolerance, SmallRye Stork, Micrometer, OpenTelemetry, JUnit 5, RestAssured, Mockito, Maven, GraalVM Native Image, Podman/Docker/Jib, Kubernetes/OpenShift/Knative extensions
