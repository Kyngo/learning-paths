---
title: "Instrumentation"
weight: 5
---

# Instrumentation

Instrumentation is the process of adding telemetry code to your applications so they emit metrics, logs, and traces. OpenTelemetry provides a unified API and SDK for all three signals. This section covers auto vs manual instrumentation, language-specific setup, and best practices.

## Auto vs Manual Instrumentation

### Auto-Instrumentation

Auto-instrumentation injects telemetry into your application without code changes. It hooks into frameworks and libraries (HTTP clients, database drivers, messaging systems) to automatically create spans, record metrics, and correlate logs.

| Aspect | Auto-Instrumentation | Manual Instrumentation |
|--------|---------------------|----------------------|
| Setup effort | Minimal — agent or SDK plugin | Requires code changes |
| Coverage | Framework-level (HTTP, DB, gRPC, messaging) | Any custom business logic |
| Customisation | Limited to configuration | Full control over spans and attributes |
| Business context | None (no domain-specific attributes) | Rich (order ID, user tier, payment method) |
| Maintenance | Library updates may change behaviour | You own the code |

**Best practice:** Use auto-instrumentation as the baseline, then add manual instrumentation for business-critical paths where you need custom attributes and spans.

### What Auto-Instrumentation Captures

Typical auto-instrumented spans for a Spring Boot application:

```text
HTTP Server: POST /api/orders (from Spring MVC)
├── JDBC: SELECT * FROM inventory WHERE sku = ? (from JDBC driver)
├── HTTP Client: POST https://payment.internal/charge (from HttpClient)
└── Kafka Producer: order-events (from Kafka client)
```

What it does **not** capture:

- Business logic timing ("time to validate order rules")
- Custom attributes ("order contained 15 items", "user is premium tier")
- Internal decision points ("selected warehouse: EU-WEST-1")

## Java Instrumentation

### Auto-Instrumentation with the Java Agent

The OpenTelemetry Java agent is a JAR that attaches to the JVM and instruments popular libraries automatically.

```bash
# Download the agent
curl -L -o opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# Run with the agent
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
  -Dotel.traces.sampler=parentbased_traceidratio \
  -Dotel.traces.sampler.arg=0.1 \
  -jar my-service.jar
```

Supported libraries (auto-instrumented):

| Category | Libraries |
|----------|----------|
| Web frameworks | Spring MVC, Spring WebFlux, JAX-RS, Servlet |
| HTTP clients | HttpClient, OkHttp, Apache HttpClient, WebClient |
| Database | JDBC, Hibernate, R2DBC, MyBatis |
| Messaging | Kafka, RabbitMQ, SQS, JMS |
| Caching | Redis (Jedis, Lettuce), Memcached |
| gRPC | gRPC client and server |

Docker configuration:

```dockerfile
FROM eclipse-temurin:21-jre
COPY opentelemetry-javaagent.jar /opt/agent/
COPY my-service.jar /app/
ENV JAVA_TOOL_OPTIONS="-javaagent:/opt/agent/opentelemetry-javaagent.jar"
ENV OTEL_SERVICE_NAME="order-service"
ENV OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"
ENTRYPOINT ["java", "-jar", "/app/my-service.jar"]
```

### Manual Instrumentation in Java

Add the OpenTelemetry API dependency:

```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
</dependency>
```

Create custom spans for business logic:

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

public class OrderService {

    private static final Tracer tracer = GlobalOpenTelemetry.getTracer("order-service");

    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("create-order")
            .setAttribute("order.item_count", request.getItems().size())
            .setAttribute("order.user_id", request.getUserId())
            .setAttribute("order.currency", request.getCurrency())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            validateOrder(request);
            Order order = processPayment(request);
            span.setAttribute("order.id", order.getId().toString());
            span.setAttribute("order.total", order.getTotal().doubleValue());
            return order;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }

    private void validateOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("validate-order")
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            // Validation logic
            span.addEvent("validation-complete",
                io.opentelemetry.api.common.Attributes.of(
                    io.opentelemetry.api.common.AttributeKey.longKey("rules_checked"), 12L
                ));
        } finally {
            span.end();
        }
    }
}
```

### Custom Metrics in Java

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.DoubleHistogram;
import io.opentelemetry.api.metrics.Meter;

public class OrderMetrics {

    private static final Meter meter = GlobalOpenTelemetry.getMeter("order-service");

    private static final LongCounter ordersCreated = meter
        .counterBuilder("orders.created")
        .setDescription("Number of orders created")
        .setUnit("{orders}")
        .build();

    private static final DoubleHistogram orderProcessingDuration = meter
        .histogramBuilder("orders.processing.duration")
        .setDescription("Time to process an order")
        .setUnit("s")
        .build();

    public void recordOrderCreated(String currency, String tier) {
        ordersCreated.add(1,
            io.opentelemetry.api.common.Attributes.of(
                io.opentelemetry.api.common.AttributeKey.stringKey("currency"), currency,
                io.opentelemetry.api.common.AttributeKey.stringKey("user_tier"), tier
            ));
    }

    public void recordProcessingDuration(double seconds, String status) {
        orderProcessingDuration.record(seconds,
            io.opentelemetry.api.common.Attributes.of(
                io.opentelemetry.api.common.AttributeKey.stringKey("status"), status
            ));
    }
}
```

## Python Instrumentation

### Auto-Instrumentation

```bash
# Install the SDK and auto-instrumentation packages
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install  # installs instrumentors for detected libraries

# Run with auto-instrumentation
opentelemetry-instrument \
  --service_name order-service \
  --exporter_otlp_endpoint http://otel-collector:4317 \
  python app.py
```

Supported libraries:

| Category | Libraries |
|----------|----------|
| Web frameworks | Flask, Django, FastAPI, Starlette |
| HTTP clients | requests, httpx, urllib3, aiohttp |
| Database | psycopg2, SQLAlchemy, PyMySQL, pymongo |
| Messaging | Kafka (confluent), Celery, boto3 (SQS) |
| Caching | redis-py |

### Manual Instrumentation in Python

```python
from opentelemetry import trace
from opentelemetry.trace import StatusCode

tracer = trace.get_tracer("order-service")

def create_order(request: CreateOrderRequest) -> Order:
    with tracer.start_as_current_span(
        "create-order",
        attributes={
            "order.item_count": len(request.items),
            "order.user_id": request.user_id,
            "order.currency": request.currency,
        },
    ) as span:
        try:
            validate_order(request)
            order = process_payment(request)
            span.set_attribute("order.id", str(order.id))
            span.set_attribute("order.total", float(order.total))
            return order
        except Exception as e:
            span.set_status(StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise


def validate_order(request: CreateOrderRequest) -> None:
    with tracer.start_as_current_span("validate-order") as span:
        # Validation logic
        span.add_event("validation-complete", {"rules_checked": 12})
```

### Custom Metrics in Python

```python
from opentelemetry import metrics

meter = metrics.get_meter("order-service")

orders_created = meter.create_counter(
    name="orders.created",
    description="Number of orders created",
    unit="{orders}",
)

order_duration = meter.create_histogram(
    name="orders.processing.duration",
    description="Time to process an order",
    unit="s",
)

def record_order_created(currency: str, tier: str) -> None:
    orders_created.add(1, {"currency": currency, "user_tier": tier})

def record_processing_duration(seconds: float, status: str) -> None:
    order_duration.record(seconds, {"status": status})
```

## Go Instrumentation

### Setup and Manual Instrumentation

Go does not have a Java-style agent, so instrumentation is always explicit. However, instrumentation libraries provide automatic spans for popular frameworks.

```go
package main

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func initTracer() (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(context.Background(),
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String("order-service"),
            attribute.String("environment", "production"),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}
```

Creating custom spans:

```go
var tracer = otel.Tracer("order-service")

func CreateOrder(ctx context.Context, req CreateOrderRequest) (Order, error) {
    ctx, span := tracer.Start(ctx, "create-order",
        trace.WithAttributes(
            attribute.Int("order.item_count", len(req.Items)),
            attribute.String("order.user_id", req.UserID),
            attribute.String("order.currency", req.Currency),
        ),
    )
    defer span.End()

    if err := validateOrder(ctx, req); err != nil {
        span.SetStatus(codes.Error, err.Error())
        span.RecordError(err)
        return Order{}, err
    }

    order, err := processPayment(ctx, req)
    if err != nil {
        span.SetStatus(codes.Error, err.Error())
        span.RecordError(err)
        return Order{}, err
    }

    span.SetAttributes(
        attribute.String("order.id", order.ID),
        attribute.Float64("order.total", order.Total),
    )
    return order, nil
}
```

**Critical Go rule:** Always pass `context.Context` through every function in the call chain. The trace context lives in the context; without it, spans cannot form parent-child relationships.

## Instrumentation Best Practices

### What to Instrument

| Priority | What | Why |
|----------|------|-----|
| P0 | HTTP server endpoints | Latency, error rate, throughput per route |
| P0 | HTTP/gRPC client calls | Downstream dependency health |
| P0 | Database queries | Slow queries, connection pool exhaustion |
| P1 | Message produce/consume | Queue lag, processing time |
| P1 | Cache operations | Hit rate, latency |
| P2 | Business-critical logic | Custom spans for order processing, payment flow |
| P2 | External API calls | Third-party reliability |

### Attribute Guidelines

```text
# Good: bounded, useful for filtering
span.set_attribute("http.method", "POST")
span.set_attribute("order.currency", "EUR")
span.set_attribute("user.tier", "premium")

# Bad: high-cardinality explosion
span.set_attribute("user.email", "user@example.com")  # PII + cardinality
span.set_attribute("request.body", "{...}")            # unbounded size
span.set_attribute("order.id", "ORD-12345")            # OK for traces, not for metrics
```

### Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Not passing context | Orphaned spans with no parent | Thread context through all calls |
| Too many spans | Performance overhead, noisy traces | Instrument at service boundaries, not every method |
| Missing `span.End()` | Memory leak, spans never exported | Use `defer span.End()` in Go, try-finally in Java |
| Sensitive data in attributes | Security/compliance risk | Never include PII, tokens, or passwords |
| No error recording | Errors invisible in traces | Always call `setStatus(ERROR)` and `recordException` |

## Key Takeaways

- Use auto-instrumentation as a baseline for framework-level telemetry (HTTP, DB, messaging), then add manual instrumentation for business-critical paths with custom attributes
- The OpenTelemetry Java agent requires zero code changes — attach it to the JVM and configure via environment variables; Python uses `opentelemetry-instrument`; Go requires explicit SDK setup
- Always pass context through the call chain — in Go via `context.Context`, in Java via `Scope`, in Python via the context manager pattern — without it, spans become orphaned
- Record business-relevant attributes on spans (order value, user tier, region) — these are what make traces useful for debugging production issues beyond what auto-instrumentation provides
- Instrument at service boundaries (P0) before internal logic (P2) — HTTP endpoints, client calls, and database queries give the most diagnostic value per line of instrumentation code
- Never include PII, tokens, or unbounded data in span attributes — keep attributes bounded and useful for filtering
