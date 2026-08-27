---
title: "Streaming & Real-Time Data"
weight: 7
---

## Why Streaming?

Batch processing assumes data can wait. Streaming processes data as it arrives — enabling fraud detection within milliseconds, real-time dashboards, live recommendations, and event-driven architectures. Any use case where latency of minutes or hours is unacceptable requires streaming.

---

## Apache Kafka

Kafka is the dominant distributed event streaming platform. It acts as a durable, high-throughput, fault-tolerant message broker that decouples producers from consumers.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | A named feed of messages (like a table for events) |
| **Partition** | A topic is split into partitions for parallelism and ordering |
| **Offset** | A sequential ID for each message within a partition |
| **Producer** | Publishes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | A set of consumers that share the work of reading a topic |
| **Broker** | A Kafka server; a cluster has multiple brokers |
| **Replication** | Each partition is replicated across brokers for fault tolerance |

### Kafka Architecture

```text
Producer ──▶ Topic: orders ──▶ Consumer Group: analytics
                │                    ├── Consumer 1 (partition 0, 1)
                │                    └── Consumer 2 (partition 2, 3)
                │
                └──▶ Consumer Group: fraud-detection
                     └── Consumer 3 (all 4 partitions)
```

**Key insight:** each consumer group independently tracks its position (offsets). Multiple groups can read the same topic at different speeds without interfering.

### Producer Configuration

```python
from confluent_kafka import Producer

config = {
    "bootstrap.servers": "kafka-1:9092,kafka-2:9092,kafka-3:9092",
    "acks": "all",                    # Wait for all replicas to acknowledge
    "enable.idempotence": True,       # Exactly-once semantics
    "compression.type": "lz4",        # Compress messages
    "linger.ms": 5,                   # Batch messages for 5ms
    "batch.size": 65536,              # 64KB batch size
}

producer = Producer(config)

def delivery_callback(err, msg):
    if err:
        print(f"Delivery failed: {err}")
    else:
        print(f"Delivered to {msg.topic()} [{msg.partition()}] @ {msg.offset()}")

# Produce a message
import json
event = {"order_id": "ORD-123", "amount": 99.95, "timestamp": "2025-06-15T10:30:00Z"}
producer.produce(
    topic="orders",
    key="ORD-123",
    value=json.dumps(event).encode("utf-8"),
    callback=delivery_callback,
)
producer.flush()
```

### Consumer Configuration

```python
from confluent_kafka import Consumer

config = {
    "bootstrap.servers": "kafka-1:9092,kafka-2:9092,kafka-3:9092",
    "group.id": "order-analytics",
    "auto.offset.reset": "earliest",  # Start from beginning if no committed offset
    "enable.auto.commit": False,       # Manual commit for exactly-once
}

consumer = Consumer(config)
consumer.subscribe(["orders"])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            print(f"Consumer error: {msg.error()}")
            continue

        event = json.loads(msg.value().decode("utf-8"))
        process_order(event)

        # Commit offset after successful processing
        consumer.commit(asynchronous=False)
finally:
    consumer.close()
```

### Delivery Guarantees

| Guarantee | How | Tradeoff |
|-----------|-----|----------|
| At-most-once | Commit offset before processing | May lose messages |
| At-least-once | Commit offset after processing | May process duplicates |
| Exactly-once | Idempotent producer + transactional consumer | Highest overhead, strictest |

**Exactly-once** requires `enable.idempotence=True` on the producer and transactional reads on the consumer. In practice, many systems use **at-least-once with idempotent consumers** (deduplication at the consumer level).

### Topic Design

| Decision | Guidance |
|----------|----------|
| Partition count | Start with 6–12 for moderate throughput; scale to hundreds for high throughput |
| Partition key | Choose a key that distributes evenly (user ID, order ID) — avoid hot keys |
| Retention | `retention.ms` — how long messages are kept (default 7 days) |
| Compaction | `cleanup.policy=compact` — keep only the latest value per key (useful for changelogs) |
| Replication factor | 3 for production (tolerates 1 broker failure) |

---

## Event Time vs Processing Time

This distinction is fundamental to correct streaming computation.

| Concept | Definition | Example |
|---------|-----------|---------|
| **Event time** | When the event actually occurred | Order placed at 14:02:03 |
| **Processing time** | When the system processes the event | Event received at 14:02:07 |
| **Ingestion time** | When the event entered the streaming platform | Kafka timestamp at 14:02:05 |

**Why it matters:** events arrive out of order and with variable delay. A 5-minute window based on processing time includes different events than the same window based on event time. For accurate analytics, use **event time**.

### Watermarks

A watermark is the system's estimate of how far behind the latest event time the stream might be. Events arriving after the watermark are considered **late**.

```text
Event times:    14:00  14:01  14:02  [gap]  14:01 (late!)  14:03
Watermark:      14:00  14:01  14:02  14:02  14:02          14:03
```

---

## Kafka Streams

Kafka Streams is a lightweight library (not a separate cluster) for building stream processing applications directly in Java/Kotlin.

```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-enrichment");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

StreamsBuilder builder = new StreamsBuilder();

// Read orders stream
KStream<String, String> orders = builder.stream("orders");

// Read customer table (compacted topic)
KTable<String, String> customers = builder.table("customers");

// Enrich orders with customer data
KStream<String, String> enriched = orders.join(
    customers,
    (order, customer) -> mergeJson(order, customer)
);

// Write to output topic
enriched.to("enriched-orders");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

---

## Apache Flink

Flink is a distributed stream processing engine designed for **stateful computations** over unbounded data. It excels at event-time processing, windowing, and exactly-once guarantees.

### Flink vs Kafka Streams vs Spark Streaming

| Aspect | Flink | Kafka Streams | Spark Structured Streaming |
|--------|-------|---------------|---------------------------|
| Architecture | Distributed cluster | Library (no cluster) | Spark cluster |
| Processing model | True streaming | True streaming | Micro-batch (mostly) |
| State management | Built-in, RocksDB-backed | Built-in, RocksDB or in-memory | Relies on checkpoint |
| Event-time support | Best-in-class | Good | Good |
| Exactly-once | Native | Native (within Kafka) | Native |
| Latency | Milliseconds | Milliseconds | Seconds (micro-batch interval) |
| Best for | Complex stateful streaming | Kafka-to-Kafka transforms | Unified batch + stream |

### Flink Python Example (PyFlink)

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment, EnvironmentSettings

env = StreamExecutionEnvironment.get_execution_environment()
t_env = StreamTableEnvironment.create(env)

# Define Kafka source
t_env.execute_sql("""
    CREATE TABLE orders (
        order_id STRING,
        customer_id STRING,
        amount DECIMAL(10, 2),
        event_time TIMESTAMP(3),
        WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'orders',
        'properties.bootstrap.servers' = 'kafka:9092',
        'properties.group.id' = 'flink-analytics',
        'format' = 'json',
        'scan.startup.mode' = 'latest-offset'
    )
""")

# 5-minute tumbling window aggregation
t_env.execute_sql("""
    SELECT
        TUMBLE_START(event_time, INTERVAL '5' MINUTE) AS window_start,
        COUNT(*) AS order_count,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY TUMBLE(event_time, INTERVAL '5' MINUTE)
""").print()
```

---

## Amazon Kinesis

Kinesis is AWS's managed streaming platform — an alternative to self-managed Kafka.

| Component | Purpose | Kafka Equivalent |
|-----------|---------|-----------------|
| Kinesis Data Streams | Ingest and buffer events | Kafka topics |
| Kinesis Data Firehose | Deliver to S3, Redshift, etc. | Kafka Connect sinks |
| Kinesis Data Analytics | SQL/Flink over streams | Kafka Streams / Flink |

**When to choose Kinesis over Kafka:**
- Already invested in AWS ecosystem
- Want fully managed (no brokers to maintain)
- Lower throughput requirements (Kafka scales further)

**When to choose Kafka:**
- Multi-cloud or hybrid deployments
- Higher throughput requirements
- Need Kafka's ecosystem (Connect, Streams, Schema Registry)
- Want to avoid AWS lock-in

---

## Windowing Patterns

Streaming aggregations use **windows** to group unbounded data into finite chunks.

| Window Type | Description | Use Case |
|------------|-------------|----------|
| **Tumbling** | Fixed-size, non-overlapping | Count orders per 5-minute window |
| **Sliding** | Fixed-size, overlapping (by slide interval) | Moving average over 10 minutes, sliding every 1 minute |
| **Session** | Gap-based — closes after inactivity | User session analytics |
| **Global** | Single window for all data | Full aggregation with custom trigger |

```text
Tumbling (5 min):  |------|------|------|------|
Sliding (10/5):    |------------|
                        |------------|
                             |------------|
Session (gap=2m):  |--events--| gap |--events--|
```

---

## Key Takeaways

1. **Apache Kafka** is the industry standard for event streaming — topics, partitions, consumer groups, and offset management are core concepts every data engineer must know
2. **Event time** (when it happened) vs **processing time** (when it was processed) is a critical distinction — use event time with watermarks for correct streaming analytics
3. **Exactly-once semantics** requires idempotent producers and transactional consumers; in practice, many systems use at-least-once with idempotent sinks
4. **Flink** excels at stateful stream processing with event-time semantics; **Kafka Streams** is ideal for lightweight Kafka-to-Kafka transforms; **Spark Structured Streaming** suits teams already on Spark
5. **Windowing** (tumbling, sliding, session) is how streaming systems group unbounded data into finite aggregations
6. **Kinesis** is a viable managed alternative to Kafka on AWS, but Kafka offers more flexibility, higher throughput, and a richer ecosystem
