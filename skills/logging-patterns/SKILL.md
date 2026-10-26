---
name: logging-patterns
description: Java logging best practices with SLF4J/JBoss Logging, structured logging (JSON), and MDC for request tracing in Quarkus. Includes AI-friendly log formats for Claude Code debugging. Use when user asks about logging, debugging application flow, or analyzing logs.
---

# Logging Patterns Skill

**Rule: JSON is the default log format.** Plain text logging is only for local development. Production, CI/CD, and Kubernetes environments must use structured JSON.

## When to Use
- User says "add logging" / "improve logs" / "debug this"
- Analyzing application flow from logs
- Setting up structured logging (JSON)
- Request tracing with correlation IDs
- AI/Claude Code needs to analyze application behavior

---

## AI-Friendly Logging

> **Key insight:** JSON logs are better for AI analysis - faster parsing, fewer tokens, direct field access.

### Why JSON for AI/Claude Code?

```
# Text format - AI must "interpret" the string
2026-01-29 10:15:30 INFO OrderService - Order 12345 created for user-789, total: 99.99

# JSON format - AI extracts fields directly
{"timestamp":"2026-01-29T10:15:30Z","level":"INFO","orderId":12345,"userId":"user-789","total":99.99}
```

| Aspect | Text | JSON |
|--------|------|------|
| Parsing | Regex/interpretation | Direct field access |
| Token usage | Higher (repeated patterns) | Lower (structured) |
| Error extraction | Parse stack trace text | `exception` field |
| Filtering | grep patterns | `jq` queries |

### Log Format Optimized for AI Analysis

```json
{
  "timestamp": "2026-01-29T10:15:30.123Z",
  "level": "INFO",
  "logger": "com.example.service.OrderService",
  "message": "Order created",
  "requestId": "req-abc123",
  "traceId": "trace-xyz",
  "orderId": 12345,
  "userId": "user-789",
  "duration_ms": 45,
  "step": "payment_completed"
}
```

**Key fields for AI debugging:**
- `requestId` - group all logs from same request
- `step` - track progress through flow
- `duration_ms` - identify slow operations
- `level` - quick filter for errors

### Reading Logs with AI/Claude Code

When asking AI to analyze logs:

```bash
# Get recent errors
cat app.log | jq 'select(.level == "ERROR")' | tail -20

# Follow specific request
cat app.log | jq 'select(.requestId == "req-abc123")'

# Find slow operations
cat app.log | jq 'select(.duration_ms > 1000)'
```

AI can then:
1. Parse JSON directly (no guessing)
2. Follow request flow via requestId
3. Identify exactly where errors occurred
4. Measure timing between steps

---

## Quick Setup (Quarkus)

### Structured JSON Logging

Quarkus does not ship JSON logging by default — add the `quarkus-logging-json` extension.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-logging-json</artifactId>
</dependency>
```

```properties
# application.properties — this is the REQUIRED default for every app
quarkus.log.console.json=true
```

### Profile-Based Switching

Quarkus config supports per-profile overrides with a `%profile.` prefix — no separate YAML documents needed.

```properties
# application.properties
# Default/production: structured JSON (for AI, CI/CD, production)
quarkus.log.console.json=true

# Dev mode: human-readable console, overrides the line above
%dev.quarkus.log.console.json=false
```

```bash
# Default: human-readable in dev mode (per the override above)
./mvnw quarkus:dev

# Force JSON even in dev, to test the production format
./mvnw quarkus:dev -Dquarkus.log.console.json=true
```

### OpenTelemetry Integration

With `quarkus-opentelemetry` on the classpath, trace and span IDs are automatically correlated with log records, so JSON log lines can be joined with traces in Jaeger, Tempo, or any OTel-compatible backend.

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-opentelemetry</artifactId>
</dependency>
```

```properties
# application.properties — tracing + structured logging together
quarkus.otel.traces.sampler=always_on
quarkus.log.console.json=true
```

### Adding Custom Fields via MDC

```java
import org.jboss.logging.MDC;

MDC.put("orderId", order.getId());
MDC.put("step", "order_created");

log.info("Order created");

// Output includes the MDC map as structured fields:
// {"message":"Order created","mdc":{"orderId":123,"step":"order_created"}, ...}
```

```properties
# Write MDC entries as root-level JSON fields instead of nesting them under "mdc"
quarkus.log.console.json.mdc.flat-fields=true
```

---

## SLF4J / JBoss Logging Basics

### Logger Declaration

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@ApplicationScoped
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    // use `log` directly for logging
}
```

Quarkus bridges SLF4J, JBoss Logging, and JUL onto a single backend, so the classic `LoggerFactory.getLogger(MyClass.class)` field works unchanged. Quarkus also offers `io.quarkus.logging.Log` — a static logging facade that needs no field declaration (`Log.info("Order created")`) — as an idiomatic shortcut if you prefer it.

### Parameterized Logging

```java
// ✅ GOOD: Evaluated only if level enabled
log.debug("Processing order {} for user {}", orderId, userId);

// ❌ BAD: Always concatenates
log.debug("Processing order " + orderId + " for user " + userId);

// ✅ For expensive operations
if (log.isDebugEnabled()) {
    log.debug("Order details: {}", order.toDetailedString());
}
```

### Exception Logging

```java
// ✅ GOOD: Pass exception as last argument
try {
    processOrder(order);
} catch (PaymentException e) {
    log.error("Payment failed for order {}", order.getId(), e);
}

// ❌ BAD: Loses stack trace
log.error("Payment failed: " + e.getMessage());

// ❌ BAD: Exception in placeholder (toString only)
log.error("Payment failed for order {}: {}", order.getId(), e);
```

### Log Levels Guide

| Level | When to use |
|-------|------------|
| `ERROR` | Something broke — action needed. Failed payment, unrecoverable state |
| `WARN` | Something unexpected but handled. Retry succeeded, fallback used |
| `INFO` | Business events. Order created, user logged in, job completed |
| `DEBUG` | Technical details. SQL queries, HTTP requests, cache hits |
| `TRACE` | Verbose internals. Method entry/exit, loop iterations |

### What NOT to Log

```java
// ❌ NEVER
log.info("Login user={} password={}", username, password);  // Credentials
log.debug("Calling API with token {}", jwt);                  // Tokens, API keys, session IDs
log.info("Payment card={}", cardNumber);                      // Full card number / CVV
log.info("Created customer {}", customer);                    // toString() may dump email, address, IBAN...
log.debug("Request body: {}", rawBody);                       // Payloads may contain any of the above

// ✅ Identify, don't expose
log.info("Login attempted userId={}", userId);
log.info("Payment authorised cardLast4={}", last4);
log.info("Created customer customerId={}", customer.id());
```

- Treat as sensitive: passwords, secrets, tokens, `Authorization`/`Cookie` headers, card data, national IDs, health data, and personal data (email, phone, address, birth date).
- Records print every component in `toString()` — log the ID, or override `toString()` on records that carry personal data.
- Keep full request/response bodies and headers out of production logs; if HTTP client logging is needed, keep it at DEBUG.

### Log Once, at the Boundary

```java
// ❌ BAD: Log-and-rethrow at every layer - the same stack trace appears three times
try {
    paymentClient.charge(order);
} catch (PaymentException e) {
    log.error("Payment failed", e);
    throw e;
}

// ✅ GOOD: Lower layers handle the exception, or wrap it with context and rethrow - no log
try {
    paymentClient.charge(order);
} catch (PaymentException e) {
    throw new OrderProcessingException("Payment failed for order " + order.getId(), e);
}
```

The boundary that turns the exception into a response logs it once, with the request context — in Quarkus that's the catch-all `@Provider` `ExceptionMapper<Exception>` (see `references/web.md` in the `quarkus` skill).

- Expected business errors (not found, validation) are WARN or DEBUG without a stack trace; only unexpected failures are ERROR with the exception.
- Messaging consumers and scheduled jobs are boundaries too: log once there, then let the retry or failure strategy act.

### Include Context

```java
// ❌ BAD: Useless when reading the logs later
log.error("Error occurred", e);

// ✅ GOOD: What failed, for which entity, at which step - exception last
log.error("Order processing failed orderId={} step={} attempt={}", orderId, "payment", attempt, e);
```

Context shared by every line of a request (request ID, user ID) belongs in MDC, set once by a filter (see below).

---

## MDC (Mapped Diagnostic Context)

MDC attaches context (request ID, user ID) to every log line in the current request automatically.

> Quarkus overrides the standard SLF4J/Logback MDC implementation for compatibility with its reactive core — always import `org.jboss.logging.MDC`, not `org.slf4j.MDC` (the latter silently stops working).

### Request Filter for Correlation IDs

```java
import org.jboss.logging.MDC;
import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ContainerRequestFilter;
import jakarta.ws.rs.container.ContainerResponseContext;
import jakarta.ws.rs.container.ContainerResponseFilter;
import jakarta.ws.rs.ext.Provider;

@Provider
public class CorrelationIdRequestFilter implements ContainerRequestFilter {

    @Override
    public void filter(ContainerRequestContext requestContext) {
        String requestId = requestContext.getHeaderString("X-Request-Id");
        if (requestId == null) {
            requestId = UUID.randomUUID().toString().substring(0, 8);
        }
        MDC.put("requestId", requestId);
    }
}

@Provider
public class CorrelationIdResponseFilter implements ContainerResponseFilter {

    @Override
    public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) {
        MDC.clear();
    }
}
```

### MDC in Service Layer

```java
@ApplicationScoped
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order createOrder(OrderRequest request) {
        MDC.put("userId", request.userId());
        MDC.put("orderId", String.valueOf(request.orderId()));

        log.info("Creating order");
        // All subsequent log lines for this request automatically include userId and orderId

        return processOrder(request);
    }
}
```

### MDC with Virtual Threads

Unlike thread-pool-based frameworks — where MDC is plain `ThreadLocal` and needs a decorator to survive a handoff to a different pooled thread — Quarkus stores MDC data in a Vert.x **duplicated context** tied to the request, not to a specific OS thread. A method annotated `@RunOnVirtualThread` inherits that duplicated context automatically, so MDC values set in a `ContainerRequestFilter` are visible inside it with no extra propagation code, even though the filter and the virtual-thread method run on different threads.

### MDC Across Threads

The duplicated context only travels with work that Quarkus manages. Inject a `ManagedExecutor` (MicroProfile Context Propagation) instead of creating your own `ExecutorService`, and MDC values from the request are available in submitted tasks:

```java
@ApplicationScoped
public class NotificationService {
    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);

    private final ManagedExecutor executor;

    public NotificationService(ManagedExecutor executor) {
        this.executor = executor;
    }

    public CompletableFuture<Void> sendConfirmationAsync(Order order) {
        // ✅ requestId/userId from the request's MDC are present in this log line
        return executor.runAsync(() -> log.info("Sending confirmation"));
    }
}
```

- ❌ `CompletableFuture.runAsync(task)` without an executor runs on the common pool, and a self-created `ExecutorService` doesn't propagate anything — MDC is lost there.
- If an unmanaged executor can't be avoided, capture the map on the calling thread and restore it in the task, clearing it in `finally` so a pooled thread doesn't leak one request's context into the next:

```java
Map<String, Object> context = new HashMap<>(MDC.getMap());
unmanagedExecutor.execute(() -> {
    context.forEach(MDC::put);
    try {
        log.info("Sending confirmation");
    } finally {
        MDC.clear();
    }
});
```

---

## Quick Reference

| Pattern | Use |
|---------|-----|
| `LoggerFactory.getLogger(MyClass.class)` | Declare logger (no `@Slf4j`) |
| `log.info("msg {}", var)` | Parameterized logging |
| `log.error("msg {}", var, exception)` | Log with stack trace |
| `org.jboss.logging.MDC.put("key", "value")` | Add context to all logs for the request |
| `MDC.clear()` | Clean up in a `ContainerResponseFilter` |
| `ManagedExecutor` | Async work that keeps the request's MDC |
| `quarkus.log.console.json=true` | Enable JSON console logging (`quarkus-logging-json`) |
| `quarkus.log.console.json.mdc.flat-fields=true` | Flatten MDC into root-level JSON fields |
