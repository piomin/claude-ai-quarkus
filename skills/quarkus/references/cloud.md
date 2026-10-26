# Cloud Native - Quarkus

## Centralized Configuration

Quarkus has no dedicated "Config Server" component — centralized configuration is handled through a few complementary mechanisms instead:

- **Per-environment config shipped with the app**: Quarkus profiles (`%dev.`, `%test.`, `%prod.` property prefixes) bundle every environment's overrides into the same `application.properties`, so there's nothing external to stand up for that case.
- **Kubernetes-native config**: `quarkus-kubernetes-config` reads `ConfigMap`/`Secret` objects directly from the Kubernetes API at startup — see `kubernetes-specialist` agent. It re-reads at the next restart, not live; Quarkus config in general is not hot-reloadable at runtime.
- **External secret stores**: `quarkus-vault` for HashiCorp Vault-backed secrets.

```properties
quarkus.kubernetes-config.enabled=true
quarkus.kubernetes-config.config-maps=my-app-config
quarkus.kubernetes-config.secrets=my-app-secrets
```

## Dynamic Configuration Refresh

`@ConfigProperty`-injected fields are resolved once, at injection time, and Quarkus has no mechanism to refresh them afterward. For a value that genuinely needs to reflect a change without a restart, inject the config API directly and re-read on every call instead of caching the value in a field:

```java
@ApplicationScoped
public class FeatureFlags {

    @Inject
    Config config;

    public boolean isFeatureEnabled() {
        // Re-evaluated on every call, unlike a cached @ConfigProperty field
        return config.getOptionalValue("app.feature.enabled", Boolean.class).orElse(false);
    }
}
```

This only reflects a genuinely new value if the underlying config source itself can change at runtime (a mutable custom `ConfigSource`, for instance) — plain environment variables and `application.properties` are fixed for the life of the process either way.

## Service Discovery & Load Balancing - SmallRye Stork

On Kubernetes, Service objects already give DNS-based discovery for free — most Quarkus services never need a client-side registry at all. For anywhere that genuinely needs client-side service discovery and load balancing (Consul, static lists, or Kubernetes endpoints without going through a Service), SmallRye Stork is the built-in service-discovery-plus-load-balancing layer:

```properties
# application.properties
quarkus.stork.order-service.service-discovery.type=consul
quarkus.stork.order-service.service-discovery.consul-host=consul
quarkus.stork.order-service.service-discovery.consul-port=8500
quarkus.stork.order-service.load-balancer.type=round-robin
```

```java
@RegisterRestClient(baseUri = "stork://order-service")
public interface OrderServiceClient {

    @GET
    @Path("/orders/{id}")
    OrderDto getOrder(@PathParam("id") Long id);
}
```

Setting the REST client's base URI scheme to `stork://<service-name>` is the only code change needed — Stork resolves the actual address and picks an instance per the configured load-balancer strategy.

## API Gateway

Quarkus has no bundled API gateway component. On Kubernetes, that role is usually filled by an Ingress controller (NGINX, etc.), a service mesh (Istio), or a dedicated API gateway product (Kong, Apache APISIX) sitting in front of the cluster — routing, rate limiting, and auth-at-the-edge concerns live there rather than in a Quarkus application.

## Fault Tolerance - SmallRye Fault Tolerance

```java
@ApplicationScoped
public class ExternalApiService {
    private static final Logger log = LoggerFactory.getLogger(ExternalApiService.class);

    @RestClient
    ExternalApiClient client;

    @CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.5, delay = 5000)
    @Retry(maxRetries = 3, delay = 1000)
    @RateLimit(value = 10, window = 1, windowUnit = ChronoUnit.SECONDS)
    @Fallback(fallbackMethod = "getFallbackData")
    public ExternalData getData(String id) {
        return client.fetchData(id);
    }

    public ExternalData getFallbackData(String id) {
        log.warn("Fallback triggered for id: {}", id);
        return new ExternalData(id, "Fallback data", LocalDateTime.now());
    }
}
```

`@CircuitBreaker`, `@Retry`, `@Fallback`, `@Timeout`, and `@Bulkhead` come from the MicroProfile Fault Tolerance spec. `@RateLimit` is a SmallRye-specific addition on top of the spec. These are primarily configured via annotation attributes rather than externalized `application.properties` blocks, though every attribute can still be overridden with a property named `<ClassName>/<methodName>/<Annotation>/<attribute>`.

## Distributed Tracing - OpenTelemetry

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-opentelemetry</artifactId>
</dependency>
```

```properties
# application.properties - OTLP export is the built-in default, no separate exporter dependency needed
quarkus.otel.traces.sampler=always_on
quarkus.otel.exporter.otlp.endpoint=http://localhost:4318
```

`quarkus-opentelemetry` ships OTLP export out of the box — there's nothing else to add for the common case.

### Custom Spans

```java
@ApplicationScoped
public class OrderService {

    @Inject
    Tracer tracer;

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public Order processOrder(OrderRequest request) {
        Span span = tracer.spanBuilder("order.process")
            .setAttribute("order.type", request.type())
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            return createOrder(request);
        } finally {
            span.end();
        }
    }
}
```

Metrics (Micrometer) and tracing (OpenTelemetry) are two separate extensions in Quarkus, each with its own API — inject `io.opentelemetry.api.trace.Tracer` directly for custom spans as shown above, and `MeterRegistry` separately for custom metrics.

## Health Checks & Metrics

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-health</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
</dependency>
```

```properties
quarkus.smallrye-health.root-path=/q/health
management.metrics.tags.application=${quarkus.application.name}
```

```java
@Readiness
@ApplicationScoped
public class ExternalServiceHealthCheck implements HealthCheck {

    @Override
    public HealthCheckResponse call() {
        boolean serviceUp = checkExternalService();

        var builder = HealthCheckResponse.named("external-service")
            .withData("timestamp", LocalDateTime.now().toString());

        return serviceUp ? builder.up().build() : builder.down().build();
    }

    private boolean checkExternalService() {
        // Check external dependency
        return true;
    }
}
```

`HealthCheck` (`org.eclipse.microprofile.health.HealthCheck`) plus the `@Liveness`/`@Readiness`/`@Startup` qualifier is the MicroProfile Health standard. Both `/q/health/live` and `/q/health/ready` (and the aggregate `/q/health`) exist with zero extra code once the extension is added; see the `kubernetes-specialist` agent for how the generated Kubernetes manifest wires these in as probes automatically, and `devops-engineer`/`security-engineer` for the Prometheus `ServiceMonitor` that gets generated alongside them.

### Custom Metrics

For business metrics, inject `MeterRegistry` and build the meters once, in the constructor:

```java
@ApplicationScoped
public class OrderMetrics {
    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer processing;

    public OrderMetrics(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created")
            .description("Orders successfully created").register(registry);
        this.ordersFailed = Counter.builder("orders.failed")
            .description("Orders that failed").register(registry);
        this.processing = Timer.builder("orders.processing.duration")
            .description("Time to process an order").register(registry);
    }

    public <T> T timeProcessing(Supplier<T> work) {
        return processing.record(work);
    }

    public void created() { ordersCreated.increment(); }
    public void failed()  { ordersFailed.increment(); }
}
```

- Declarative alternative: `@Counted` / `@Timed` (`io.micrometer.core.annotation`) on CDI bean methods.
- Keep tag values low-cardinality (status, type) — never user IDs, order IDs, or raw paths.
- Meter names use dots; the Prometheus registry converts them (`orders.created` → `orders_created_total`).

## Kubernetes Deployment

Quarkus's `quarkus-kubernetes`/`quarkus-openshift`/`quarkus-knative` extensions **generate** the manifests from the project's code, config, and classpath at build time — including health probes, a Prometheus `ServiceMonitor`, and even a migration `Job` for Flyway/Liquibase, with no YAML to hand-write for the common case. Build and deploy is a single Maven property, not a separate `kubectl apply` step. Full details, properties, and examples live in the `kubernetes-specialist` agent — this is intentionally not duplicated here.

## Virtual Threads

See the `java-modern` skill for `@RunOnVirtualThread` — there is no single blanket switch for virtual threads; they're opted into per blocking endpoint/method.

## HTTP Interface Clients

See `web.md` for the full `@RegisterRestClient` + `@RestClient` pattern, Quarkus's one declarative HTTP client mechanism.

## GraalVM Native Image

Native compilation is Quarkus's headline feature, not an opt-in add-on bolted on afterward — every project generated by `quarkus create app` already ships a `native` Maven profile.

```bash
# Build a native executable (requires GraalVM locally, or use the container build below)
./mvnw package -Dnative

# Build inside a container - no local GraalVM install needed (see devops-engineer agent)
./mvnw package -Dnative -Dquarkus.native.container-build=true -Dquarkus.native.container-runtime=podman
```

Native executables start in tens of milliseconds and run with a small fraction of the memory footprint of the same app in JVM mode — the trade-off is a notably longer build, and any reflection-based code (some JSON/serialization libraries, certain CDI tricks) needs an explicit `@RegisterForReflection` hint since GraalVM can't discover it by scanning bytecode at build time the way the JVM does at runtime.

```java
// A class reached only via reflection (Class.forName, polymorphic Jackson by class name, ...)
@RegisterForReflection
public record LegacyPayload(String field) {}

// Third-party classes you cannot annotate
@RegisterForReflection(targets = { com.thirdparty.Payload.class, com.thirdparty.Header.class })
public class ReflectionConfig {}
```

```properties
# Classpath resources read at runtime must be included in the native image explicitly
quarkus.native.resources.includes=templates/**,certs/*.pem
```

DTOs consumed and returned by REST resources are registered automatically — registration is only for types reached reflectively. Run `@QuarkusIntegrationTest` against the native build to catch what was missed (see the `testing-patterns` skill).

## Graceful Shutdown

```properties
# application.properties
quarkus.shutdown.timeout=30s
```

Quarkus already waits briefly for in-flight requests to finish by default on shutdown. `quarkus.shutdown.timeout` caps how long that default wait is allowed to run.

```yaml
# Kubernetes - preStop hook to allow deregistration before the process receives SIGTERM
spec:
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
```

## Container Images & Local Kubernetes Development

Building images (Podman/Docker/Jib extensions) and the build→deploy loop (`quarkus.kubernetes.deploy=true`, and `quarkus:remote-dev` as the live-coding loop) are covered end-to-end by the `devops-engineer` and `kubernetes-specialist` agents — not repeated here.

## Quick Reference

| Component | Purpose |
|-----------|---------|
| **Profiles (`%dev.`/`%test.`/`%prod.`)** | Per-environment config bundled with the app, no Config Server |
| **Stork** | Service discovery and client-side load balancing |
| **SmallRye Fault Tolerance** | `@CircuitBreaker`, `@Retry`, `@Fallback`, `@RateLimit` |
| **OpenTelemetry** | Distributed tracing with built-in OTLP export |
| **SmallRye Health** | `/q/health/live`, `/q/health/ready` — zero extra code |
| **Micrometer + Prometheus** | `/q/metrics` |
| **Virtual Threads** | `@RunOnVirtualThread` per blocking endpoint/method |
| **REST Client** | `@RegisterRestClient`/`@RestClient` — one mechanism for declarative HTTP clients |
| **GraalVM Native Image** | Millisecond startup, low memory — built into every generated project |
| **quarkus-kubernetes/-openshift/-knative** | Generated manifests, not hand-written YAML |
| **Dev Services** | Zero-config Testcontainers for local dev and tests |
