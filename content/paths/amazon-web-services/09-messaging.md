---
title: "Messaging"
weight: 9
---

## Why Messaging?

Direct synchronous communication between services creates tight coupling — if service B is down, service A fails too. Messaging decouples services by introducing a buffer between them.

```mermaid
flowchart LR
    subgraph Synchronous["Synchronous (coupled)"]
        A1["Service A"] -->|"HTTP call"| B1["Service B"]
        B1 -->|"If down → A fails"| A1
    end
    
    subgraph Asynchronous["Asynchronous (decoupled)"]
        A2["Service A"] -->|"Send message"| Q["Queue/Topic"]
        Q -->|"Consume when ready"| B2["Service B"]
    end
```

---

## SQS (Simple Queue Service)

Fully managed message queue — producers send messages, consumers poll and process them.

### Queue Types

| Feature | Standard | FIFO |
|---------|----------|------|
| Throughput | Unlimited | 3,000 msg/s (with batching) |
| Ordering | Best-effort | Strict FIFO |
| Delivery | At-least-once (possible duplicates) | Exactly-once |
| Use case | High throughput, order doesn't matter | Order matters, no duplicates |

### How SQS Works

```mermaid
sequenceDiagram
    participant Producer
    participant SQS as SQS Queue
    participant Consumer
    
    Producer->>SQS: SendMessage("process order #123")
    Note over SQS: Message stored (up to 14 days)
    
    Consumer->>SQS: ReceiveMessage (long poll)
    SQS-->>Consumer: Message (invisible to others)
    Note over Consumer: Process message
    Consumer->>SQS: DeleteMessage
    Note over SQS: Message permanently removed
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Visibility Timeout** | Time message is hidden after being received (default: 30s) |
| **Dead Letter Queue (DLQ)** | Failed messages go here after N retries |
| **Long Polling** | Consumer waits up to 20s for messages (reduces empty responses) |
| **Message Retention** | 1 minute to 14 days (default: 4 days) |
| **Max Message Size** | 256 KB (use S3 for larger payloads) |
| **Delay Queue** | Messages invisible for N seconds after being sent |

### Dead Letter Queue Pattern

```mermaid
flowchart LR
    Producer["Producer"] --> Main["Main Queue"]
    Main --> Consumer["Consumer"]
    Consumer -->|"Process fails<br>(3 attempts)"| DLQ["Dead Letter Queue"]
    DLQ --> Alert["CloudWatch Alarm"]
    DLQ --> Manual["Manual Investigation"]
```

---

## SNS (Simple Notification Service)

Pub/sub messaging — one message published to a topic is delivered to all subscribers.

```mermaid
flowchart TD
    Publisher["Publisher"] --> Topic["SNS Topic:<br>order-events"]
    Topic --> Sub1["SQS Queue<br>(inventory service)"]
    Topic --> Sub2["Lambda<br>(email notification)"]
    Topic --> Sub3["HTTP Endpoint<br>(analytics)"]
    Topic --> Sub4["SQS Queue<br>(shipping service)"]
```

### SNS Subscriber Types

| Subscriber | Use Case |
|-----------|----------|
| SQS Queue | Decouple processing |
| Lambda | Serverless processing |
| HTTP/HTTPS | Webhooks |
| Email | Notifications |
| SMS | Alerts |
| Mobile Push | App notifications |

### SNS + SQS (Fan-Out Pattern)

One event triggers multiple independent processors:

```mermaid
flowchart TD
    Order["Order Placed"] --> SNS["SNS: order-placed"]
    SNS --> Q1["SQS: inventory-updates"]
    SNS --> Q2["SQS: email-notifications"]
    SNS --> Q3["SQS: analytics-events"]
    
    Q1 --> S1["Inventory Service"]
    Q2 --> S2["Email Service"]
    Q3 --> S3["Analytics Service"]
```

Each service processes independently — if email service is slow, inventory isn't affected.

---

## EventBridge

Event bus for building event-driven architectures. More powerful than SNS for complex routing.

```mermaid
flowchart LR
    Sources["Event Sources"]
    Sources --> EB["EventBridge Bus"]
    EB -->|"Rule: source=orders<br>detail-type=OrderPlaced"| T1["Target: Lambda"]
    EB -->|"Rule: source=orders<br>detail-type=OrderFailed"| T2["Target: SQS"]
    EB -->|"Rule: source=*<br>(catch-all)"| T3["Target: CloudWatch Logs"]
```

### EventBridge vs SNS

| Feature | SNS | EventBridge |
|---------|-----|-------------|
| Filtering | Basic (attribute-based) | Rich (JSON content-based) |
| Schema | None | Schema registry + discovery |
| Sources | Your services | AWS services + SaaS + custom |
| Targets | Subscribers | 20+ AWS service integrations |
| Archive/Replay | No | Yes (replay past events) |
| Use case | Simple fan-out | Complex event routing |

### Event Structure

```json
{
  "source": "com.myapp.orders",
  "detail-type": "OrderPlaced",
  "detail": {
    "orderId": "12345",
    "customerId": "cust-789",
    "total": 99.99,
    "items": [{"sku": "ABC", "qty": 2}]
  },
  "time": "2024-01-15T10:30:00Z"
}
```

---

## Choosing a Messaging Service

| Need | Service | Why |
|------|---------|-----|
| Decouple producer/consumer | SQS | Simple queue, at-least-once delivery |
| Strict ordering | SQS FIFO | Guaranteed order, exactly-once |
| One-to-many broadcast | SNS | Fan-out to multiple subscribers |
| Complex event routing | EventBridge | Content-based filtering, schema registry |
| Fan-out + independent processing | SNS + SQS | Each consumer has its own queue |
| Event replay/archive | EventBridge | Replay past events for debugging |
| Streaming (high throughput) | Kinesis | Real-time data streams, ordered |

---

## Patterns

### Request-Response via Queue

```mermaid
flowchart LR
    Client["Client"] -->|"Request + ReplyTo queue"| ReqQ["Request Queue"]
    ReqQ --> Worker["Worker"]
    Worker -->|"Response"| RespQ["Response Queue"]
    RespQ --> Client
```

### Saga Pattern (Distributed Transactions)

```mermaid
flowchart LR
    Order["Create Order"] -->|"Event"| Payment["Process Payment"]
    Payment -->|"Success"| Ship["Ship Order"]
    Payment -->|"Failure"| Compensate["Cancel Order<br>(compensating action)"]
```

---

## Key Takeaways

1. **SQS for decoupling** — buffer between services, handles backpressure
2. **SNS for fan-out** — one event, multiple consumers
3. **EventBridge for complex routing** — content-based filtering, schema registry
4. **Always use DLQs** — failed messages need investigation, not silent loss
5. **Long polling reduces cost** — fewer empty ReceiveMessage calls
6. **SNS + SQS for independent processing** — each consumer processes at its own pace
7. **FIFO when order matters** — but accept the throughput tradeoff
