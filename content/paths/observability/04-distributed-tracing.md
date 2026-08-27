---
title: "Distributed Tracing"
weight: 4
---

# Distributed Tracing

Distributed tracing follows individual requests as they propagate through multiple services. When a user hits "Place Order" and that request touches an API gateway, order service, inventory service, payment service, and notification service, a trace captures the entire journey — showing where time is spent, which calls failed, and how services depend on each other.

## Core Concepts

### Traces and Spans

A **trace** represents the end-to-end journey of a single request. It consists of one or more **spans**.

A **span** represents a single unit of work within that trace: an HTTP request, a database query, a message published to a queue.

```text
Trace ID: abc-123-def-456
│
├── Span A: api-gateway (root span)
│   ├── Span B: auth-service → validate-token
│   ├── Span C: order-service → create-order
│   │   ├── Span D: inventory-service → check-stock
│   │   ├── Span E: payment-service → charge
│   │   └── Span F: order-service → save-to-db
│   └── Span G: notification-service → send-email
```

Each span contains:

| Field | Description | Example |
|-------|-------------|---------|
| `trace_id` | Globally unique ID for the trace | `abc123def456` |
| `span_id` | Unique ID for this span | `span-001` |
| `parent_span_id` | The span that initiated this one | `span-000` (root has none) |
| `operation_name` | What work was performed | `POST /api/orders` |
| `start_time` | When the span began | `2026-08-27T08:15:32.004Z` |
| `duration` | How long the span took | `245ms` |
| `status` | OK, ERROR, or UNSET | `ERROR` |
| `attributes` | Key-value metadata | `http.method=POST`, `http.status_code=201` |
| `events` | Timestamped annotations within the span | Exception thrown at T+120ms |

### Trace Context Propagation

For tracing to work across services, each service must propagate the trace context (trace ID + span ID) to downstream calls. This happens via HTTP headers.

#### W3C Trace Context (Standard)

The W3C Trace Context specification defines two headers:

```http
traceparent: 00-abc123def456789012345678-0123456789abcdef-01
tracestate: vendor1=value1,vendor2=value2
```

The `traceparent` header format:

```text
version-trace_id-parent_span_id-trace_flags
00     -abc123..-0123456789ab..-01

version:        00 (always)
trace_id:       32 hex characters (16 bytes)
parent_span_id: 16 hex characters (8 bytes)
trace_flags:    01 = sampled, 00 = not sampled
```

#### B3 Propagation (Zipkin)

Zipkin uses a different header format:

```http
X-B3-TraceId: abc123def456789012345678
X-B3-SpanId: 0123456789abcdef
X-B3-ParentSpanId: fedcba9876543210
X-B3-Sampled: 1
```

Or the compact single-header format:

```http
b3: abc123def456789012345678-0123456789abcdef-1-fedcba9876543210
```

#### AWS X-Ray

```http
X-Amzn-Trace-Id: Root=1-abc12345-def6789012345678;Parent=0123456789ab;Sampled=1
```

**Best practice:** Use W3C Trace Context. It's the industry standard and supported by all modern tracing backends. OpenTelemetry uses it by default.

## OpenTelemetry

OpenTelemetry (OTel) is the CNCF project that provides a vendor-neutral standard for telemetry. It defines APIs, SDKs, and the OpenTelemetry Collector for metrics, logs, and traces.

### Architecture

```text
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Application    │     │   Application    │     │   Application    │
│  (OTel SDK)      │     │  (OTel SDK)      │     │  (OTel SDK)      │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │ OTLP                   │ OTLP                   │ OTLP
         └────────────┬───────────┘────────────────────────┘
                      │
               ┌──────▼──────┐
               │   OTel      │
               │  Collector  │
               │ (pipeline)  │
               └──────┬──────┘
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      ┌───────┐  ┌────────┐  ┌──────┐
      │ Jaeger│  │ Grafana│  │Datadog│
      │       │  │ Tempo  │  │      │
      └───────┘  └────────┘  └──────┘
```

### OTLP (OpenTelemetry Protocol)

OTLP is the wire protocol for sending telemetry data. It supports gRPC and HTTP/protobuf:

```text
# gRPC (default, most efficient)
endpoint: otel-collector:4317

# HTTP/protobuf
endpoint: http://otel-collector:4318/v1/traces
```

### The OTel Collector

The Collector is a proxy that receives, processes, and exports telemetry data. It has three pipeline stages:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  attributes:
    actions:
      - key: environment
        value: production
        action: upsert

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  otlp/tempo:
    endpoint: tempo:4317

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, attributes]
      exporters: [otlp/jaeger, otlp/tempo]
```

## Tracing Backends

### Jaeger

Open-source, CNCF-graduated tracing backend. Originally developed by Uber.

| Feature | Details |
|---------|---------|
| Storage | Elasticsearch, Cassandra, Kafka, Badger (local), gRPC plugin |
| Query | Web UI, gRPC API |
| Sampling | Adaptive, probabilistic, rate-limiting |
| Deployment | All-in-one binary for dev, microservices for production |

Quick start for development:

```bash
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/jaeger:2.0 \
  --set receivers.otlp.protocols.grpc.endpoint=0.0.0.0:4317 \
  --set receivers.otlp.protocols.http.endpoint=0.0.0.0:4318
```

UI available at `http://localhost:16686`.

### Grafana Tempo

Grafana's tracing backend. Designed for massive scale with minimal infrastructure.

| Feature | Details |
|---------|---------|
| Storage | Object storage (S3, GCS, Azure Blob) — no indexing database needed |
| Query | TraceQL (powerful trace query language), Grafana integration |
| Cost | Very low — uses object storage instead of Elasticsearch/Cassandra |
| Trade-off | Trace lookup by ID is fast; search by attributes requires Tempo's search features |

TraceQL example:

```text
# Find traces where an HTTP span returned 500 and took more than 2 seconds
{ span.http.status_code = 500 && duration > 2s }

# Find traces through the payment service with errors
{ resource.service.name = "payment-service" && status = error }
```

### AWS X-Ray

AWS's native tracing service. Integrated with Lambda, API Gateway, ECS, EKS.

| Feature | Details |
|---------|---------|
| Integration | Native with AWS services (no SDK needed for Lambda) |
| Sampling | Centralised sampling rules |
| Service Map | Automatic topology visualisation |
| Pricing | Pay per trace recorded and scanned |

X-Ray SDK adds trace context automatically for AWS SDK calls, HTTP clients, and SQL queries.

### Zipkin

One of the earliest open-source tracing systems (Twitter, 2012).

| Feature | Details |
|---------|---------|
| Storage | Elasticsearch, Cassandra, MySQL, in-memory |
| Protocol | B3 propagation headers (also supports W3C) |
| Maturity | Stable, widely supported, large ecosystem |
| Trade-off | Fewer features than Jaeger/Tempo; simpler to operate |

### Comparison

| Feature | Jaeger | Tempo | X-Ray | Zipkin |
|---------|--------|-------|-------|--------|
| License | Apache 2.0 | AGPLv3 | Proprietary | Apache 2.0 |
| Storage | ES/Cassandra | Object storage | AWS-managed | ES/Cassandra |
| Query language | Basic filters | TraceQL | Filter expressions | Basic filters |
| Sampling | Adaptive | Head/tail | Centralised rules | Probabilistic |
| Best for | General purpose | Grafana users, scale | AWS-native workloads | Simple deployments |

## Sampling

At scale, storing every trace is prohibitively expensive. Sampling strategies reduce volume while preserving signal.

### Head-Based Sampling

The decision to sample is made at the start of the trace (at the root span). Common strategies:

```text
# Probabilistic: sample 10% of traces
probability: 0.1

# Rate-limiting: max 100 traces per second
rate_limit: 100
```

**Problem:** You don't know at trace start whether the request will be interesting (an error, slow, etc.). A 10% sample rate means you miss 90% of errors.

### Tail-Based Sampling

The decision is made after the trace completes, based on its characteristics:

```yaml
# OTel Collector tail sampling
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 2000}
      - name: baseline
        type: probabilistic
        probabilistic: {sampling_percentage: 5}
```

This keeps **all** error traces, **all** slow traces, and a 5% baseline sample. Much more useful for debugging.

**Trade-off:** Tail sampling requires buffering complete traces in memory before deciding, which increases collector resource requirements.

## Span Attributes and Semantic Conventions

OpenTelemetry defines semantic conventions for common span attributes:

```text
# HTTP spans
http.request.method = "POST"
http.response.status_code = 201
url.full = "https://api.example.com/orders"
server.address = "api.example.com"

# Database spans
db.system = "postgresql"
db.statement = "SELECT * FROM orders WHERE id = $1"
db.operation.name = "SELECT"

# Messaging spans
messaging.system = "kafka"
messaging.destination.name = "order-events"
messaging.operation.type = "publish"
```

Following semantic conventions ensures consistent attribute names across services and languages, enabling cross-service queries.

## Key Takeaways

- A trace represents one request's journey through your system; it consists of spans, each representing a unit of work — the parent-child relationship between spans reveals service dependencies
- W3C Trace Context is the standard propagation format — use it unless you have a legacy Zipkin (B3) dependency
- OpenTelemetry provides vendor-neutral instrumentation; the OTel Collector decouples your applications from your tracing backend
- Tail-based sampling is superior to head-based for debugging because it retains all errors and slow traces rather than sampling blindly
- Jaeger and Tempo are the leading open-source backends; Tempo is significantly cheaper at scale due to object storage, while Jaeger offers more mature query features
- Semantic conventions for span attributes ensure consistency across services — always use the standard attribute names for HTTP, database, and messaging spans
