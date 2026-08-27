---
title: "Logging"
weight: 3
---

# Logging

Logs are timestamped, immutable records of discrete events. They provide the richest context of any telemetry signal — the actual error message, the request payload that triggered it, the stack trace. This section covers structured logging, log levels, JSON format, aggregation pipelines, and the major log management stacks.

## Unstructured vs Structured Logging

### The Problem with Unstructured Logs

Traditional logs are free-form text:

```text
2026-08-27 08:15:32 ERROR OrderService - Failed to process order ORD-9182 for user 4521: PaymentGatewayTimeout after 30042ms
```

This is human-readable but machine-hostile. To extract the order ID, user ID, or duration, you need regex parsing — which is fragile, slow, and breaks when the message format changes.

### Structured Logging

Structured logs emit events as key-value pairs (typically JSON):

```json
{
  "timestamp": "2026-08-27T08:15:32.004Z",
  "level": "ERROR",
  "logger": "com.example.OrderService",
  "message": "Failed to process order",
  "service": "order-service",
  "environment": "production",
  "trace_id": "abc123def456",
  "span_id": "789ghi012",
  "order_id": "ORD-9182",
  "user_id": "4521",
  "payment_gateway": "stripe",
  "duration_ms": 30042,
  "error_type": "PaymentGatewayTimeout",
  "stack_trace": "com.example.PaymentClient.charge(PaymentClient.java:142)..."
}
```

Every field is independently queryable, filterable, and aggregatable. You can answer questions like "show me all errors from `stripe` gateway in the last hour" without regex.

### Comparison

| Aspect | Unstructured | Structured |
|--------|-------------|-----------|
| Human readability | High (in terminal) | Medium (JSON is verbose) |
| Machine parsability | Low (requires regex) | High (native key-value) |
| Query flexibility | Limited (full-text search) | Full (filter by any field) |
| Schema evolution | Fragile (format changes break parsers) | Resilient (add fields without breaking) |
| Storage efficiency | Lower (redundant text) | Higher (compresses well) |
| Aggregation | Difficult | Native (group by any field) |

## Log Levels

Log levels indicate the severity of an event. Use them consistently across all services.

| Level | When to Use | Examples |
|-------|------------|---------|
| `TRACE` | Fine-grained debugging; disabled in production | Method entry/exit, variable values |
| `DEBUG` | Diagnostic information for development | SQL queries executed, cache hits/misses |
| `INFO` | Normal business events | Order created, user logged in, deploy started |
| `WARN` | Unexpected but recoverable conditions | Retry attempt, deprecated API usage, slow query |
| `ERROR` | Failures requiring attention | Unhandled exception, downstream timeout, data corruption |
| `FATAL` | Application cannot continue | Out of memory, missing critical config, database unreachable at startup |

### Rules for Effective Log Levels

1. **Production services run at `INFO`** — `DEBUG` and `TRACE` are disabled unless actively investigating
2. **`WARN` does not mean "soon to be an error"** — it means something unexpected happened but the system recovered
3. **`ERROR` means action is needed** — every ERROR log should either trigger an alert or be investigated
4. **If everything is `INFO`, nothing is** — be disciplined about severity classification

## Structured Logging in Practice

### Java (SLF4J + Logback)

```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>trace_id</includeMdcKeyName>
      <includeMdcKeyName>span_id</includeMdcKeyName>
      <customFields>{"service":"order-service","environment":"${ENV}"}</customFields>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="JSON" />
  </root>
</configuration>
```

```java
@Slf4j
public class OrderService {
    public Order createOrder(CreateOrderRequest request) {
        log.info("Creating order: userId={}, items={}", request.getUserId(), request.getItemCount());
        try {
            Order order = processOrder(request);
            log.info("Order created: orderId={}, total={}", order.getId(), order.getTotal());
            return order;
        } catch (PaymentException e) {
            log.error("Payment failed: userId={}, gateway={}, errorCode={}",
                request.getUserId(), e.getGateway(), e.getCode(), e);
            throw e;
        }
    }
}
```

### Python (structlog)

```python
import structlog

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.JSONRenderer(),
    ],
)

logger = structlog.get_logger()

def create_order(request: CreateOrderRequest) -> Order:
    logger.info("creating_order", user_id=request.user_id, item_count=len(request.items))
    try:
        order = process_order(request)
        logger.info("order_created", order_id=order.id, total=order.total)
        return order
    except PaymentError as e:
        logger.error("payment_failed", user_id=request.user_id, gateway=e.gateway, exc_info=True)
        raise
```

### Go (log/slog)

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
    slog.SetDefault(logger)
}

func createOrder(req CreateOrderRequest) (Order, error) {
    slog.Info("creating order", "user_id", req.UserID, "item_count", len(req.Items))

    order, err := processOrder(req)
    if err != nil {
        slog.Error("order creation failed", "user_id", req.UserID, "error", err)
        return Order{}, err
    }

    slog.Info("order created", "order_id", order.ID, "total", order.Total)
    return order, nil
}
```

## Log Aggregation

Individual log files on individual servers are useless at scale. Log aggregation centralises logs from all services into a queryable store.

### The Aggregation Pipeline

```text
Application ──→ Log Shipper ──→ Message Queue ──→ Processor ──→ Storage ──→ UI
(emit)          (collect)       (buffer)          (parse/enrich) (index)    (query)
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

The original open-source log aggregation stack:

| Component | Role | Notes |
|-----------|------|-------|
| **Elasticsearch** | Storage and search engine | Full-text and structured queries |
| **Logstash** | Log processing pipeline | Parse, transform, enrich, route |
| **Kibana** | Visualisation and exploration | Dashboards, KQL queries, saved searches |
| **Beats** (Filebeat) | Lightweight log shipper | Installed on each host, tails log files |

Pipeline:

```text
App → stdout → Filebeat → Logstash → Elasticsearch → Kibana
```

Filebeat configuration:

```yaml
# filebeat.yml
filebeat.inputs:
  - type: container
    paths:
      - /var/lib/docker/containers/*/*.log
    json.keys_under_root: true
    json.add_error_key: true

output.logstash:
  hosts: ["logstash:5044"]
```

### EFK Stack (Elasticsearch, Fluentd/Fluent Bit, Kibana)

Common in Kubernetes environments. Replaces Logstash with Fluentd or Fluent Bit:

| Component | Role | Advantage over Logstash |
|-----------|------|------------------------|
| **Fluent Bit** | Lightweight log shipper | 450 KB binary, minimal memory footprint |
| **Fluentd** | Log processor/router | Plugin ecosystem, Kubernetes-native |

Fluent Bit Kubernetes configuration:

```yaml
# fluent-bit.conf
[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            docker
    Mem_Buf_Limit     5MB

[FILTER]
    Name              kubernetes
    Match             kube.*
    Merge_Log         On
    K8S-Logging.Parser On

[OUTPUT]
    Name              es
    Match             *
    Host              elasticsearch
    Port              9200
    Index             logs-%Y.%m.%d
    Type              _doc
```

### CloudWatch Logs (AWS)

AWS's native log service. No infrastructure to manage.

```text
Application → CloudWatch Agent → Log Group → Log Stream → Insights Query
```

CloudWatch Logs Insights query language:

```text
# Error rate by service in the last hour
fields @timestamp, @message, service, level
| filter level = "ERROR"
| stats count() as error_count by service
| sort error_count desc

# P99 latency from structured logs
fields @timestamp, duration_ms, endpoint
| filter endpoint = "/api/orders"
| stats pct(duration_ms, 99) as p99, avg(duration_ms) as avg_duration by bin(5m)

# Find all logs for a specific trace
fields @timestamp, @message
| filter trace_id = "abc123def456"
| sort @timestamp asc
```

## Log Correlation

The most powerful technique in log management is correlating logs with traces. Every log line should include `trace_id` and `span_id`:

```json
{
  "timestamp": "2026-08-27T08:15:32.004Z",
  "level": "ERROR",
  "trace_id": "abc123def456",
  "span_id": "789ghi012",
  "message": "Payment gateway timeout"
}
```

This lets you:

1. See a spike in errors on a dashboard
2. Click a trace ID to see the full distributed trace
3. From the trace, see all correlated logs across all services involved in that request

### Logging Anti-Patterns

| Anti-Pattern | Why It's Bad | Correct Approach |
|-------------|-------------|-----------------|
| Logging sensitive data (passwords, tokens, PII) | Security and compliance violation | Mask or exclude sensitive fields |
| Logging inside tight loops | Floods log storage, degrades performance | Log at loop boundaries, aggregate counts |
| Using `print()` or `System.out.println()` | No levels, no structure, no correlation | Use a logging framework with JSON output |
| Catch-and-log-and-throw | Duplicate log entries at every layer | Log at the boundary where the error is handled |
| Missing context (bare "error occurred") | Useless for debugging | Include IDs, durations, parameters |
| Logging entire request/response bodies | Storage explosion, potential PII leak | Log summary fields, not full payloads |

## Key Takeaways

- Structured logging (JSON with key-value fields) is mandatory for production systems — it enables machine-parsable queries, filtering, and aggregation
- Use log levels consistently: `INFO` for business events, `WARN` for recoverable issues, `ERROR` for actionable failures — never leave `DEBUG` enabled in production
- Always include `trace_id` and `span_id` in every log line to enable correlation with distributed traces
- ELK (Elasticsearch/Logstash/Kibana), EFK (Elasticsearch/Fluentd/Kibana), and CloudWatch Logs are the three most common aggregation stacks — EFK is preferred for Kubernetes
- Log aggregation follows a pipeline: emit → collect → buffer → process → store → query — each stage can be independently scaled
- Never log sensitive data, full request bodies, or inside tight loops — these are the most common logging mistakes that cause cost and compliance issues
