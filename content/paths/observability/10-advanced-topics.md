---
title: "Advanced Topics"
weight: 10
---

# Advanced Topics

Once you have metrics, logs, traces, dashboards, and alerting in place, advanced observability techniques push further — answering questions that traditional telemetry cannot. This section covers continuous profiling, eBPF, synthetic monitoring, chaos engineering observability, and AIOps.

## Continuous Profiling

Traditional profiling captures a snapshot of CPU, memory, or I/O usage during a debugging session. Continuous profiling runs *always* in production, attaching profiling data to traces to explain *why* a span is slow — not just *that* it's slow.

### What Profiling Captures

| Profile Type | What It Shows | Example Insight |
|-------------|---------------|-----------------|
| CPU | Which functions consume CPU time | `JsonSerializer.serialize()` takes 40% of CPU in checkout |
| Heap (Alloc) | Which functions allocate the most memory | `String.concat()` in logging creates 2 GB/min of garbage |
| Wall Clock | Real elapsed time (including I/O waits) | `HttpClient.send()` blocks for 800ms waiting on downstream |
| Lock/Mutex | Contention on locks | `ConcurrentHashMap.put()` contention under high concurrency |
| I/O | Blocking I/O operations | `FileInputStream.read()` blocking 200ms on cold disk |

### How It Works

Continuous profilers sample call stacks at a fixed frequency (typically 100 Hz) and aggregate them into flame graphs:

```text
100% ──── handleRequest()
├── 35% ── validateOrder()
│   ├── 20% ── JsonSchema.validate()
│   └── 15% ── RegexMatcher.match()
├── 45% ── processPayment()
│   ├── 40% ── HttpClient.send()    ← 40% of request time waiting on payment API
│   └──  5% ── PaymentMapper.map()
└── 20% ── saveOrder()
    ├── 15% ── JdbcTemplate.update()
    └──  5% ── AuditLogger.log()
```

### Profiling Tools

| Tool | Type | Languages | Backend |
|------|------|-----------|---------|
| **Pyroscope** (Grafana) | Open source | Go, Java, Python, Ruby, .NET, Rust, Node.js | Self-hosted or Grafana Cloud |
| **Datadog Continuous Profiler** | SaaS | Java, Python, Go, .NET, Ruby, Node.js, PHP | Datadog |
| **Parca** | Open source (CNCF) | Any (eBPF-based, no agent per language) | Self-hosted |
| **async-profiler** | Library | Java/JVM | Local or integrate with backends |

### Pyroscope Integration with Go

```go
import "github.com/grafana/pyroscope-go"

func main() {
    pyroscope.Start(pyroscope.Config{
        ApplicationName: "order-service",
        ServerAddress:   "http://pyroscope:4040",
        ProfileTypes: []pyroscope.ProfileType{
            pyroscope.ProfileCPU,
            pyroscope.ProfileAllocObjects,
            pyroscope.ProfileAllocSpace,
            pyroscope.ProfileInuseObjects,
            pyroscope.ProfileInuseSpace,
            pyroscope.ProfileGoroutines,
            pyroscope.ProfileMutexCount,
            pyroscope.ProfileMutexDuration,
            pyroscope.ProfileBlockCount,
            pyroscope.ProfileBlockDuration,
        },
        Tags: map[string]string{
            "environment": "production",
            "region":      "eu-west-1",
        },
    })
    // ... application code
}
```

### Trace-to-Profile Correlation

The most powerful profiling feature: link a slow trace span directly to its flame graph. When a trace shows `processPayment()` took 2 seconds, click through to see *exactly* which function calls consumed that time.

Grafana supports this natively with Tempo + Pyroscope integration — span profiling attaches profile data to individual trace spans.

## eBPF Observability

eBPF (Extended Berkeley Packet Filter) is a Linux kernel technology that allows running sandboxed programs inside the kernel without modifying kernel source or loading kernel modules. For observability, this means collecting telemetry with near-zero overhead and without modifying applications.

### What eBPF Can Observe

| Domain | What eBPF Sees | Traditional Alternative |
|--------|---------------|----------------------|
| Network | Every TCP connection, DNS lookup, HTTP request | tcpdump, application-level metrics |
| File System | Every file open, read, write, delete | Audit logs, inotify |
| Process | System calls, context switches, scheduling | strace (high overhead), /proc |
| Security | Unexpected syscalls, process execution, file access | SELinux audit, seccomp |

### eBPF Observability Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| **Cilium** | Kubernetes networking + observability | L3/L4/L7 visibility, service map, Hubble UI |
| **Pixie** (CNCF) | Auto-instrumented Kubernetes observability | No code changes, captures HTTP/gRPC/DNS/DB |
| **Grafana Beyla** | eBPF-based auto-instrumentation | HTTP/gRPC metrics and traces with zero code |
| **bpftrace** | Ad-hoc kernel tracing | DTrace-like scripting for Linux |
| **Parca** | eBPF-based continuous profiling | Language-agnostic CPU profiling |

### Grafana Beyla Example

Beyla uses eBPF to automatically instrument HTTP and gRPC services without any SDK:

```yaml
# beyla-config.yaml
open_port: 8080
otel_metrics_export:
  endpoint: http://otel-collector:4318/v1/metrics
otel_traces_export:
  endpoint: http://otel-collector:4318/v1/traces
attributes:
  kubernetes:
    enable: true
```

Deploy as a sidecar or DaemonSet — it intercepts syscalls to observe HTTP requests, producing metrics and traces without touching your application code.

### bpftrace One-Liners

```bash
# Count system calls by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Histogram of read() sizes
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args->ret > 0/ { @bytes = hist(args->ret); }'

# Track TCP connections by destination port
bpftrace -e 'kprobe:tcp_connect { @[args->sk->__sk_common.skc_dport] = count(); }'

# Latency of DNS lookups
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:getaddrinfo { @start[tid] = nsecs; }
             uretprobe:/lib/x86_64-linux-gnu/libc.so.6:getaddrinfo { @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'
```

### When to Use eBPF

| Use Case | Why eBPF |
|----------|----------|
| Observing services you cannot modify (third-party, legacy) | No code changes required |
| Network-level visibility (who talks to whom) | Sees all TCP/UDP, not just instrumented calls |
| Ultra-low-overhead profiling | Kernel-level, < 1% overhead |
| Security monitoring | Sees syscalls, file access, process execution |
| Kubernetes service mesh alternative | L7 visibility without sidecar proxy overhead |

## Synthetic Monitoring

Synthetic monitoring runs scripted tests against your production systems at regular intervals — simulating user behaviour to detect issues before real users encounter them.

### Types of Synthetic Tests

| Type | What It Tests | Frequency | Example |
|------|--------------|-----------|---------|
| **HTTP ping** | Endpoint availability and latency | Every 30–60s | `GET /health` returns 200 in < 500ms |
| **API test** | Full API workflow | Every 1–5min | Create order → get order → verify fields |
| **Browser test** | End-to-end user journey | Every 5–15min | Login → search → add to cart → checkout |
| **SSL/TLS check** | Certificate validity | Every 1h | Cert expires in > 30 days |
| **DNS check** | DNS resolution | Every 1min | `api.example.com` resolves correctly |

### Grafana Synthetic Monitoring

```yaml
# k6 synthetic test
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  frequency: '60s',
  locations: ['eu-west-1', 'us-east-1', 'ap-southeast-1'],
};

export default function () {
  const res = http.get('https://api.example.com/health');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'body contains ok': (r) => r.body.includes('ok'),
  });
}
```

### Synthetic vs Real User Monitoring (RUM)

| Aspect | Synthetic | RUM |
|--------|-----------|-----|
| Traffic | Scripted, consistent | Real user behaviour |
| Coverage | Predefined paths only | All user journeys |
| Availability signal | Detect outages before users | Detect issues users actually hit |
| Geographic coverage | Configurable test locations | Wherever users are |
| Baseline | Stable (same test, same conditions) | Variable (device, network, behaviour) |

**Best practice:** Use both. Synthetics detect outages during low-traffic periods (midnight); RUM catches issues that only affect specific devices or user segments.

## Chaos Engineering Observability

Chaos engineering deliberately introduces failures to test system resilience. Observability is essential — without it, you can't measure the impact of injected failures.

### The Chaos-Observability Loop

```text
1. Hypothesis    → "If payment-service fails, checkout degrades gracefully"
2. Observe       → Record baseline SLIs (error rate, latency, throughput)
3. Inject fault  → Kill payment-service pods / inject 500ms latency / drop 50% of packets
4. Observe       → Monitor SLIs during the experiment
5. Analyse       → Compare actual behaviour to hypothesis
6. Improve       → Fix gaps (missing circuit breaker, no fallback, bad timeout)
```

### Common Chaos Experiments

| Experiment | What It Tests | Observability Signal |
|-----------|---------------|---------------------|
| Kill a pod/instance | Auto-recovery, load balancing | Error rate, recovery time |
| Inject latency (500ms) | Timeout configuration, circuit breakers | P99 latency, downstream impact |
| Drop network packets (50%) | Retry logic, connection handling | Retry rate, error rate |
| Fill disk to 95% | Disk space alerting, log rotation | Disk usage alerts, application errors |
| Exhaust CPU (stress-ng) | Autoscaling, throttling | CPU metrics, request queuing |
| Corrupt DNS responses | DNS failover, caching | DNS error metrics, resolution time |

### Chaos Tools

| Tool | Type | Integration |
|------|------|-------------|
| **Litmus** | Kubernetes-native, CNCF | ChaosEngine CRDs, Prometheus metrics |
| **Chaos Mesh** | Kubernetes-native, CNCF | CRDs, Grafana dashboard |
| **AWS Fault Injection Service** | AWS-managed | CloudWatch integration |
| **Gremlin** | SaaS | Datadog, PagerDuty integration |
| **Toxiproxy** | Proxy-based (Shopify) | Inject latency/errors at network level |

## AIOps and Machine Learning

AIOps applies machine learning to observability data to automate detection, correlation, and remediation.

### Anomaly Detection

Instead of static thresholds ("alert if error rate > 1%"), ML models learn normal patterns and alert on deviations:

```text
Normal pattern: Error rate is 0.05% on weekdays, 0.02% on weekends, spikes to 0.3% during deploys
Static threshold: Alert if > 1%    → Misses the 0.3% anomaly during non-deploy hours
Anomaly detection: Alert if > 3σ from expected value → Catches 0.3% if no deploy occurred
```

| Approach | Method | When to Use |
|----------|--------|-------------|
| Static threshold | Fixed value (e.g., > 1%) | Well-understood, stable metrics |
| Dynamic threshold | Moving average + standard deviation | Metrics with predictable patterns |
| Seasonal model | Accounts for daily/weekly cycles | Traffic metrics, business metrics |
| Clustering | Groups similar anomalies together | Reducing alert noise across many services |

### Automated Root Cause Analysis

Advanced platforms correlate multiple signals to suggest root cause:

```text
Alert: Error rate increased on order-service
Correlated signals:
  - Deploy v2.14.3 occurred 3 minutes before
  - DB connection pool utilisation jumped to 100%
  - payment-service latency increased 10x
  - No infrastructure changes detected

Suggested root cause: Deploy v2.14.3 introduced a regression
affecting database performance. Recommend rollback.
```

### Current State of AIOps

| What Works Today | What's Aspirational |
|-----------------|---------------------|
| Anomaly detection on individual metrics | Multi-signal causal analysis |
| Alert grouping and deduplication | Fully automated remediation |
| Log pattern clustering | Natural language incident investigation |
| Forecast-based capacity alerts | Self-healing systems |

**Realistic assessment:** AIOps is most valuable for reducing alert noise (clustering, deduplication, anomaly detection) and suggesting correlations. Fully automated root cause analysis and remediation remain research problems for most environments.

## Observability Pipeline Optimisation

As telemetry volume grows, controlling cost and performance becomes critical.

### Cost Reduction Strategies

| Strategy | Savings | Trade-off |
|----------|---------|-----------|
| Tail-based trace sampling | 80–95% trace volume reduction | Requires collector memory for buffering |
| Log level filtering at collector | 50–70% log volume reduction | Lose DEBUG/TRACE logs |
| Metric aggregation (recording rules) | Reduce query load | Pre-computed views only |
| Log archival (cold storage) | 90% storage cost reduction | Query latency for archived logs |
| Attribute dropping (remove noisy fields) | 10–30% per-event size reduction | Lose some diagnostic detail |

### OpenTelemetry Collector Pipeline for Cost Control

```yaml
processors:
  # Filter out health check spans
  filter/healthcheck:
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
        - 'attributes["http.route"] == "/ready"'

  # Drop DEBUG logs
  filter/loglevel:
    logs:
      log_record:
        - 'severity_number < 9'  # Drop below INFO

  # Remove noisy attributes
  attributes/cleanup:
    actions:
      - key: http.user_agent
        action: delete
      - key: http.request.header.cookie
        action: delete

  # Tail-based sampling
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow
        type: latency
        latency: {threshold_ms: 2000}
      - name: sample
        type: probabilistic
        probabilistic: {sampling_percentage: 5}

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [filter/healthcheck, tail_sampling, attributes/cleanup, batch]
      exporters: [otlp/tempo]
    logs:
      receivers: [otlp]
      processors: [filter/loglevel, attributes/cleanup, batch]
      exporters: [otlp/loki]
```

## Key Takeaways

- Continuous profiling answers the "why is this slow?" question that traces cannot — it links flame graphs to trace spans, showing exactly which functions consume time and resources
- eBPF enables zero-instrumentation observability by hooking into the kernel; tools like Cilium, Pixie, and Beyla provide network, application, and profiling visibility without SDK changes
- Synthetic monitoring detects outages before users do — run HTTP, API, and browser tests from multiple geographies at regular intervals; combine with RUM for full coverage
- Chaos engineering requires strong observability: you must measure baseline SLIs before injecting failures, monitor during the experiment, and compare actual behaviour to your hypothesis
- AIOps is most practically useful today for anomaly detection, alert deduplication, and log clustering — fully automated root cause analysis remains aspirational for most organisations
- Telemetry cost control is an ongoing concern: use tail-based sampling, log level filtering, attribute pruning, and cold storage archival to reduce volume by 80%+ without losing diagnostic value
