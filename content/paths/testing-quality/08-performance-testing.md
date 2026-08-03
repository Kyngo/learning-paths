---
title: "Performance Testing"
weight: 8
---

# Performance Testing

Performance testing validates that a system meets its speed, scalability, and stability requirements under expected and extreme conditions. It identifies bottlenecks before they reach production and provides data to support capacity planning and SLO validation.

---

## Types of Performance Tests

| Type | Purpose | Duration | Load Pattern |
|------|---------|----------|--------------|
| **Load test** | Validate behavior under expected load | 5–30 min | Ramp to target, hold steady |
| **Stress test** | Find breaking point | 15–60 min | Ramp beyond capacity |
| **Soak test** | Detect memory leaks, resource exhaustion | 2–24 hours | Constant moderate load |
| **Spike test** | Validate auto-scaling and recovery | 5–15 min | Sudden burst, then drop |
| **Breakpoint test** | Find maximum capacity | Variable | Incremental ramp until failure |
| **Benchmark** | Establish baseline for comparison | 1–5 min | Fixed load, repeatable |

---

## Load Testing Tools Comparison

| Feature | k6 | Locust | JMeter | Gatling |
|---------|------|--------|--------|---------|
| Language | JavaScript | Python | XML/GUI | Scala/Java |
| Protocol | HTTP, gRPC, WebSocket | HTTP, custom | HTTP, JDBC, JMS, FTP | HTTP, WebSocket |
| Scripting | Code-first | Code-first | GUI + code | Code-first (DSL) |
| Resource usage | Low (Go runtime) | Medium (Python) | High (Java) | Medium (JVM) |
| Distributed | Built-in (k6 Cloud) | Built-in (master/worker) | JMeter Remote | Distributed by design |
| Reporting | CLI + JSON + Grafana | Web UI | HTML/XML reports | HTML reports |
| Learning curve | Low | Low | Medium | Medium |
| Best for | CI/CD integration, developers | Python teams, custom protocols | Enterprise, complex scenarios | High-throughput HTTP |

---

## k6 — Developer-Friendly Load Testing

### Basic Load Test

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // Ramp up to 50 users
    { duration: '3m', target: 50 },   // Hold at 50 users
    { duration: '1m', target: 100 },  // Ramp to 100 users
    { duration: '3m', target: 100 },  // Hold at 100 users
    { duration: '1m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<300', 'p(99)<500'],  // SLO validation
    http_req_failed: ['rate<0.01'],                  // <1% error rate
  },
};

export default function () {
  const res = http.get('https://api.example.com/users');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 300ms': (r) => r.timings.duration < 300,
    'body has users': (r) => JSON.parse(r.body).length > 0,
  });

  sleep(1); // Think time between requests
}
```

### Running k6

```bash
# Local run
k6 run load-test.js

# With custom output
k6 run --out json=results.json load-test.js

# With environment variables
k6 run -e API_URL=https://staging.example.com load-test.js
```

---

## Locust — Python-Based Load Testing

```python
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)  # Think time: 1-3 seconds

    def on_start(self):
        """Login on start."""
        self.client.post("/login", json={
            "username": "testuser",
            "password": "testpass"
        })

    @task(3)  # Weight: 3x more likely than other tasks
    def view_items(self):
        self.client.get("/api/items")

    @task(1)
    def create_item(self):
        self.client.post("/api/items", json={
            "name": "Test Item",
            "price": 29.99
        })

    @task(2)
    def view_item_detail(self):
        item_id = 42  # In practice, pick from a pool
        self.client.get(f"/api/items/{item_id}")
```

### Running Locust

```bash
# Start with web UI
locust -f locustfile.py --host=https://api.example.com

# Headless mode (for CI)
locust -f locustfile.py --headless -u 100 -r 10 --run-time 5m \
  --host=https://api.example.com
```

---

## Stress Testing

Push beyond expected capacity to find the breaking point and observe degradation patterns.

```javascript
// k6 stress test
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Normal load
    { duration: '2m', target: 200 },   // High load
    { duration: '2m', target: 500 },   // Very high load
    { duration: '2m', target: 1000 },  // Breaking point?
    { duration: '5m', target: 0 },     // Recovery
  ],
};
```

**What to observe:**
- At what load does latency degrade significantly?
- Does the system recover after load drops?
- Are errors graceful (429, 503) or catastrophic (timeouts, crashes)?
- Does auto-scaling trigger and at what threshold?

---

## Soak Testing

Run moderate load for an extended period to detect slow resource leaks.

```javascript
// k6 soak test
export const options = {
  stages: [
    { duration: '5m', target: 50 },     // Ramp up
    { duration: '8h', target: 50 },     // Sustained load
    { duration: '5m', target: 0 },      // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
  },
};
```

**Detects:**
- Memory leaks (gradually increasing memory usage)
- Connection pool exhaustion
- File descriptor leaks
- Database connection leaks
- Log file growth filling disk

---

## Benchmarking

Establish repeatable baselines for isolated components:

```python
# Python benchmarking with pytest-benchmark
import pytest

def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

def test_fibonacci_benchmark(benchmark):
    result = benchmark(fibonacci, 20)
    assert result == 6765
```

### HTTP Endpoint Benchmarking

```bash
# Using hey (simple HTTP benchmarker)
hey -n 10000 -c 100 -m GET https://api.example.com/health

# Using wrk (scriptable benchmarker)
wrk -t12 -c400 -d30s https://api.example.com/users
```

---

## Profiling & Identifying Bottlenecks

### Types of Profiling

| Profiler Type | Measures | Tools |
|---------------|----------|-------|
| CPU profiler | Time spent in functions | cProfile, py-spy, async-profiler |
| Memory profiler | Allocations, heap usage | memray, tracemalloc, Valgrind |
| I/O profiler | Disk and network waits | strace, eBPF |
| Database profiler | Query execution time | EXPLAIN ANALYZE, slow query log |
| Network profiler | Latency, throughput | tcpdump, Wireshark |

### Python CPU Profiling

```python
import cProfile
import pstats

# Profile a function
profiler = cProfile.Profile()
profiler.enable()
result = expensive_function()
profiler.disable()

# View results
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # Top 20 functions
```

### Common Bottleneck Patterns

| Symptom | Likely Cause | Investigation |
|---------|-------------|---------------|
| High latency, low CPU | I/O bound (DB, network) | Profile I/O waits, check query plans |
| High CPU, high latency | CPU bound (computation) | CPU profiler, algorithmic complexity |
| Gradual degradation | Resource leak | Monitor memory/connections over time |
| Sudden degradation | Connection pool exhaustion | Check pool metrics, max connections |
| Inconsistent latency | GC pauses or lock contention | GC logs, thread dumps |

---

## SLO Validation

Performance tests should validate your Service Level Objectives:

```javascript
// k6 thresholds map directly to SLOs
export const options = {
  thresholds: {
    // Latency SLOs
    'http_req_duration{endpoint:search}': ['p95<200'],
    'http_req_duration{endpoint:checkout}': ['p99<1000'],

    // Availability SLO
    'http_req_failed': ['rate<0.001'],  // 99.9% success rate

    // Throughput SLO
    'http_reqs': ['rate>100'],  // At least 100 RPS sustained
  },
};
```

### SLO Metrics to Test

| SLO Type | Metric | Example Target |
|----------|--------|----------------|
| Latency | P50, P95, P99 response time | P95 < 200ms |
| Availability | Error rate | < 0.1% errors |
| Throughput | Requests per second | > 500 RPS |
| Saturation | Resource utilization under load | CPU < 70% at peak |

---

## Performance Budgets

Define acceptable performance boundaries and enforce them in CI:

```javascript
// k6 budget enforcement
export const options = {
  thresholds: {
    // Page load budget
    'http_req_duration{type:page}': ['p95<2000'],

    // API response budget
    'http_req_duration{type:api}': ['p95<300'],

    // Payload size budget (custom metric)
    'response_size': ['p95<50000'],  // 50KB max
  },
};

// Custom metric for response size
import { Trend } from 'k6/metrics';
const responseSize = new Trend('response_size');

export default function () {
  const res = http.get(url);
  responseSize.add(res.body.length);
}
```

### Budget Categories

| Category | Budget | Rationale |
|----------|--------|-----------|
| Time to First Byte | < 200ms | Server responsiveness |
| API response (simple) | < 100ms P95 | User-perceived speed |
| API response (complex) | < 500ms P95 | Acceptable for dashboards |
| Search queries | < 300ms P95 | User expectation from Google |
| File uploads | < 5s for 10MB | Practical limit |
| WebSocket connect | < 500ms | Real-time feature readiness |

---

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Performance Tests
on:
  pull_request:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: tests/performance/load-test.js
          flags: --out json=results.json

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results.json
```

### When to Run Performance Tests

| Trigger | Test Type | Duration |
|---------|-----------|----------|
| Every PR | Smoke test (minimal load) | 1–2 min |
| Nightly | Full load test | 15–30 min |
| Pre-release | Stress + soak test | 2–8 hours |
| Post-deploy | Baseline comparison | 5 min |

---

## Reporting & Analysis

### Key Metrics to Report

| Metric | What It Tells You |
|--------|------------------|
| P50 (median) | Typical user experience |
| P95 | Experience for most users |
| P99 | Worst case for 99% of users |
| Max | Absolute worst case (often outlier) |
| Error rate | System reliability under load |
| Throughput (RPS) | System capacity |
| Concurrent users | How many simultaneous users supported |

### Comparing Results

```
Baseline (main):    P50=45ms  P95=120ms  P99=340ms  Errors=0.02%
This PR:            P50=48ms  P95=180ms  P99=520ms  Errors=0.05%
                    (+7%)     (+50%)     (+53%)     (+150%)
```

Flag regressions: if P95 increases by more than 20%, the PR requires investigation.

---

## Key Takeaways

- **Load testing validates SLOs** — run against realistic traffic patterns, not just peak numbers
- **k6** is the best choice for developer-centric, CI-integrated testing; **Locust** wins for Python teams needing custom protocols
- **Stress tests find the ceiling**, soak tests find leaks — both are essential for production readiness
- **Profile before optimizing** — measure where time is actually spent, don't guess
- **Performance budgets in CI** prevent regressions from reaching production — treat them like test failures
- **P95 and P99 matter more than averages** — averages hide the experience of your worst-served users
- Run smoke performance tests on **every PR**, full suites **nightly** or pre-release
- Always test with **realistic data volumes and user patterns** — empty databases give misleading results
