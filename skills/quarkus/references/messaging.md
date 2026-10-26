# Messaging - Kafka with SmallRye Reactive Messaging

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-messaging-kafka</artifactId>
</dependency>
<!-- tests -->
<dependency>
    <groupId>io.smallrye.reactive</groupId>
    <artifactId>smallrye-reactive-messaging-in-memory</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <scope>test</scope>
</dependency>
```

## Producing, Consuming, Processing

```java
@ApplicationScoped
public class OrderMessaging {
    private static final Logger log = LoggerFactory.getLogger(OrderMessaging.class);

    private final Emitter<OrderEvent> orderEmitter;
    private final OrderService orderService;

    public OrderMessaging(@Channel("orders-out") Emitter<OrderEvent> orderEmitter,
                          OrderService orderService) {
        this.orderEmitter = orderEmitter;
        this.orderService = orderService;
    }

    // Imperative code → Kafka. The returned stage completes when the broker acks.
    public CompletionStage<Void> publish(OrderEvent event) {
        return orderEmitter.send(event);
    }

    // Kafka → handler. A plain (non-Uni) signature runs on a worker thread.
    @Incoming("orders-in")
    public void consume(OrderEvent event) {
        log.info("Processing order event orderId={} status={}", event.orderId(), event.status());
        orderService.apply(event);
    }

    // Stream processor: in → transform → out
    @Incoming("raw-events")
    @Outgoing("enriched-events")
    public OrderEvent enrich(OrderEvent raw) {
        return new OrderEvent(raw.orderId(), raw.status(), Instant.now());
    }
}

public record OrderEvent(String orderId, String status, Instant occurredAt) {}
```

`@Channel`, `Emitter`, `@Incoming` and `@Outgoing` come from `org.eclipse.microprofile.reactive.messaging`. The `Emitter` is injected through the constructor like any other bean.

- `@Incoming` methods that write to the database with `@Transactional` must be blocking: a plain signature, or `@Blocking` on a `Uni`-returning method. Never block the event loop.
- `@Broadcast` (`io.smallrye.reactive.messaging.annotations.Broadcast`) on an `@Outgoing` method lets several `@Incoming` methods in the same application consume that channel. Without it, a channel has a single subscriber.
- Kafka delivers at least once, so make handlers idempotent. Choose a failure strategy per channel: `dead-letter-queue`, `ignore` or `fail` (the default).
- For publishing reliably together with a database write, see the Outbox pattern in the `design-patterns` skill.

```properties
# Channel → topic. Quarkus auto-detects JSON (de)serializers for records.
mp.messaging.outgoing.orders-out.topic=orders
mp.messaging.incoming.orders-in.topic=orders
mp.messaging.incoming.orders-in.group.id=order-service
mp.messaging.incoming.orders-in.failure-strategy=dead-letter-queue
mp.messaging.incoming.raw-events.topic=raw-order-events
mp.messaging.outgoing.enriched-events.topic=enriched-order-events

# Dev/test: Kafka Dev Services starts a broker - don't set bootstrap servers there
%prod.kafka.bootstrap.servers=${KAFKA_BOOTSTRAP_SERVERS}
```

With a single connector on the classpath, `connector=smallrye-kafka` is inferred and can be omitted. Kafka Dev Services follows the same rules as the other Dev Services (see `testing.md`).

## Testing with `InMemoryConnector` + Awaitility

Switch the channels to in-memory for fast tests that need no broker. For end-to-end checks against the real broker, keep Kafka Dev Services and consume with `KafkaCompanion` (`quarkus-test-kafka-companion`).

```java
public class InMemoryChannelsResource implements QuarkusTestResourceLifecycleManager {

    @Override
    public Map<String, String> start() {
        Map<String, String> props = new HashMap<>();
        props.putAll(InMemoryConnector.switchIncomingChannelsToInMemory("orders-in", "raw-events"));
        props.putAll(InMemoryConnector.switchOutgoingChannelsToInMemory("orders-out", "enriched-events"));
        return props;
    }

    @Override
    public void stop() {
        InMemoryConnector.clear();
    }
}
```

```java
@QuarkusTest
@WithTestResource(InMemoryChannelsResource.class)
class OrderMessagingTest {

    @Inject
    @Any
    InMemoryConnector connector;

    @Test
    @DisplayName("Should enrich raw order events")
    void shouldEnrichRawOrderEvents() {
        InMemorySource<OrderEvent> source = connector.source("raw-events");
        InMemorySink<OrderEvent> sink = connector.sink("enriched-events");

        source.send(new OrderEvent("order-123", "CREATED", null));

        await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            assertThat(sink.received()).hasSize(1);
            assertThat(sink.received().getFirst().getPayload().occurredAt()).isNotNull();
        });
    }
}
```

- Never use `Thread.sleep` for async assertions; use Awaitility with a bounded `atMost`.
- Clear sinks between tests (`sink.clear()`) when several tests share a channel.
- `@InjectMock` the downstream service (e.g. `OrderService`) to assert what a consumer did.

## Quick Reference

| Annotation / API | Purpose |
|------------------|---------|
| `@Channel("name") Emitter<T>` | Send messages from imperative code |
| `@Incoming("name")` | Consume a channel |
| `@Outgoing("name")` | Produce to a channel (combine with `@Incoming` for a processor) |
| `@Broadcast` | Let several consumers in the app subscribe to one channel |
| `mp.messaging.[incoming\|outgoing].<channel>.*` | Map a channel to a topic and tune the connector |
| `InMemoryConnector` | Swap channels to in-memory in tests |
| `KafkaCompanion` | Produce/consume against the Dev Services broker in tests |
