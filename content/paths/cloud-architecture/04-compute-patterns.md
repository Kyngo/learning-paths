---
title: "Compute Patterns"
weight: 4
---

# Compute Patterns

Choosing the right compute model is one of the highest-leverage architecture decisions you'll make. It affects cost, operational complexity, scaling behaviour, and team velocity. This section provides frameworks for making that decision and patterns for getting the most out of each model.

---

## Compute Decision Framework

Every workload has different characteristics. Match the compute model to the workload, not the other way around.

```
What is the workload profile?
│
├── Stateless request/response, runs < 15 min
│   ├── < 1M requests/day → FaaS (Lambda / Cloud Functions)
│   └── > 1M requests/day → Containers (cost-optimised)
│
├── Long-running process, always-on
│   ├── Needs OS-level control → VMs
│   └── Doesn't need OS-level control → Containers
│
├── Batch processing, can tolerate interruption
│   └── Spot/preemptible instances (VMs or containers)
│
├── GPU/ML training or inference
│   ├── Training → GPU VMs or managed ML (SageMaker, Vertex AI)
│   └── Inference → GPU containers or serverless inference
│
└── Legacy app, can't be modified
    └── VMs (lift and shift)
```

### Compute Model Comparison

| Factor | VMs | Containers | FaaS |
|--------|-----|-----------|------|
| **Startup time** | Minutes | Seconds | Milliseconds (warm) to seconds (cold) |
| **Scaling granularity** | Per VM | Per container | Per request |
| **Min cost at idle** | VM cost 24/7 | Cluster cost 24/7 | Zero (scale to zero) |
| **Max control** | Full OS access | User-space control | Code only |
| **Operational burden** | Patching, hardening, monitoring | Orchestration, image management | Near-zero ops |
| **State handling** | Local disk available | Ephemeral by design | Stateless |
| **Cold start** | N/A (always running) | N/A (always running) | 100ms–10s (varies by runtime) |
| **Cost at scale** | Lower per-unit (reserved) | Lower per-unit (bin-packing) | Higher per-unit (per-invocation) |

---

## Auto-Scaling Patterns

Auto-scaling adjusts compute capacity based on demand. The goal is to match capacity to load with minimal lag and minimal waste.

### Scaling Strategies

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Target tracking** | Maintain a metric at a target value (e.g., CPU at 60%) | Steady, predictable traffic |
| **Step scaling** | Add/remove capacity in steps based on alarm thresholds | Multi-threshold responses |
| **Scheduled scaling** | Pre-scale at known times (e.g., 08:00 business hours) | Predictable daily/weekly patterns |
| **Predictive scaling** | ML-based forecasting from historical data | Recurring patterns with lead time |
| **Queue-based scaling** | Scale based on queue depth | Async workloads, batch processing |

### Scaling Architecture

```
                     Metric Source
                    (CloudWatch / Prometheus / Custom)
                          │
                          ▼
                  ┌───────────────┐
                  │  Scaling       │
                  │  Policy        │
                  │                │
                  │  Target: CPU   │
                  │  at 60%        │
                  └───────┬───────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
         ┌────────┐  ┌────────┐  ┌────────┐
         │ Inst 1 │  │ Inst 2 │  │ Inst 3 │   ← Current: 3 instances
         └────────┘  └────────┘  └────────┘

    CPU spikes to 85% → Policy triggers scale-out

         ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
         │ Inst 1 │  │ Inst 2 │  │ Inst 3 │  │ Inst 4 │  │ Inst 5 │
         └────────┘  └────────┘  └────────┘  └────────┘  └────────┘
                                                  ↑ new       ↑ new
```

### Scaling Design Rules

1. **Scale out fast, scale in slow** — add capacity immediately, remove it after a cooldown to avoid flapping
2. **Don't scale on a single metric** — combine CPU, memory, request latency, and queue depth
3. **Set minimum instances ≥ 2** — one instance is a single point of failure, even with auto-scaling
4. **Pre-warm for known events** — if you know Black Friday is coming, schedule capacity in advance
5. **Test your scaling** — run load tests to verify that scaling triggers fire correctly and fast enough

### Cooldown and Stabilisation

| Problem | Solution |
|---------|----------|
| Scale-out triggers too often (flapping) | Cooldown period: wait 300s before next scale action |
| Scale-in removes too many instances | Scale-in protection on instances processing long jobs |
| New instances receive traffic before ready | Health check grace period: don't count new instances until healthy |
| Scaling is too slow for spikes | Predictive scaling + minimum instance count |

---

## Spot and Preemptible Strategies

Spot instances (AWS), preemptible VMs (GCP), and spot VMs (Azure) offer 60–90% discounts compared to on-demand pricing. The trade-off: the provider can reclaim them with short notice (2 minutes on AWS, 30 seconds on GCP).

### Spot Architecture Patterns

| Pattern | Description | Use Case |
|---------|------------|----------|
| **Diversified fleet** | Spread across multiple instance types and AZs | Reduce interruption probability |
| **Checkpointing** | Save state periodically; resume on new instance | Batch processing, ML training |
| **Queue-worker** | Workers pull from queue; interruption = message returns to queue | Event processing, ETL |
| **Mixed fleet** | Base capacity on-demand + burst capacity on spot | Web tier with variable traffic |
| **Spot-only batch** | Entire batch runs on spot; retry on interruption | Overnight batch jobs, CI/CD |

### Mixed Fleet Architecture

```
┌──────────────────────────────────────────────────┐
│                  Auto Scaling Group               │
│                                                    │
│  Base (On-Demand): 4 instances                    │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐                     │
│  │ OD │ │ OD │ │ OD │ │ OD │     ← Always-on     │
│  └────┘ └────┘ └────┘ └────┘       baseline       │
│                                                    │
│  Burst (Spot): 0–8 instances                      │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐              │
│  │ SP │ │ SP │ │ SP │ │ SP │ │ SP │  ← Elastic   │
│  └────┘ └────┘ └────┘ └────┘ └────┘    burst      │
│                                                    │
│  Types: m5.xlarge, m5a.xlarge, m6i.xlarge,        │
│         m6a.xlarge (diversified)                   │
└──────────────────────────────────────────────────┘
```

### Spot Interruption Handling

```
Spot Interruption Notice (2 min warning)
        │
        ├── 1. Stop accepting new requests (deregister from LB)
        ├── 2. Drain in-flight requests (connection draining)
        ├── 3. Checkpoint state to durable storage (S3/EBS)
        ├── 4. Signal "unhealthy" to orchestrator
        └── 5. Replacement instance launches automatically
```

---

## Right-Sizing Methodology

Over-provisioning wastes money. Under-provisioning degrades performance. Right-sizing is the continuous process of matching instance types to actual usage.

### Right-Sizing Process

| Step | Action | Tool |
|------|--------|------|
| 1. Collect metrics | CPU, memory, network, disk I/O over 14+ days | CloudWatch, Prometheus, Datadog |
| 2. Identify waste | Instances consistently below 20% CPU / 40% memory | Compute Optimizer, Cost Explorer |
| 3. Recommend changes | Map current usage to a smaller instance type | Rightsizing recommendations |
| 4. Test in pre-prod | Run load tests on the recommended type | JMeter, k6, Locust |
| 5. Apply and monitor | Deploy the change, watch for regressions | Dashboards, alerts on P95 latency |
| 6. Repeat quarterly | Workloads change; re-evaluate every quarter | Scheduled review |

### Instance Family Selection

| Workload Type | Instance Family (AWS) | Characteristics |
|--------------|----------------------|-----------------|
| General purpose | m6i, m7g (Graviton) | Balanced CPU/memory |
| Compute-intensive | c6i, c7g | High CPU-to-memory ratio |
| Memory-intensive | r6i, r7g | High memory-to-CPU ratio |
| Storage-intensive | i3, d3 | Local NVMe, high IOPS |
| GPU / ML | p4, g5, inf2 | GPU or inference accelerator |
| Burstable | t3, t4g | Low baseline, burst on demand |

**Graviton / ARM:** ARM-based instances (AWS Graviton, Azure Ampere, GCP Tau T2A) offer 20–40% better price/performance for most workloads. Default to ARM unless your software requires x86.

---

## GPU and ML Compute

Machine learning workloads have unique compute requirements: massive parallelism for training, low-latency inference for serving.

### Training vs Inference

| Characteristic | Training | Inference |
|---------------|----------|-----------|
| Duration | Hours to days | Milliseconds per request |
| GPU utilisation | High (80–100%) | Variable (often low) |
| Batch size | Large | Single or small batch |
| Scaling | Fixed cluster | Auto-scale with traffic |
| Spot viable? | Yes (with checkpointing) | Risky (latency-sensitive) |

### Inference Architecture Options

| Option | Latency | Cost | Best For |
|--------|---------|------|----------|
| Dedicated GPU instance (24/7) | Lowest | Highest | Sustained high-volume inference |
| Auto-scaled GPU fleet | Low | Medium | Variable but always-present demand |
| Serverless inference (SageMaker, Vertex) | Medium (cold start) | Pay-per-request | Intermittent or unpredictable demand |
| CPU inference (optimised model) | Medium | Lowest | Models small enough for CPU (ONNX, TFLite) |

---

## Container Orchestration Decisions

If you choose containers, you still need to decide *how* to run them.

| Option | Ops Burden | Control | Cost Model |
|--------|-----------|---------|-----------|
| Self-managed Kubernetes | High | Full | EC2/VM cost + your time |
| Managed Kubernetes (EKS/GKE/AKS) | Medium | High | Cluster fee + node cost |
| Managed containers (ECS/Cloud Run/ACA) | Low | Medium | Per-task or per-request |
| Serverless containers (Fargate/Cloud Run) | Minimal | Low | Per-vCPU-second |

### Decision Guide

```
Do you need Kubernetes-specific features (CRDs, operators, service mesh)?
├── Yes → Managed Kubernetes (EKS, GKE, AKS)
└── No
    ├── Do you need to run on spot/preemptible nodes?
    │   ├── Yes → Managed Kubernetes with spot node pools
    │   └── No
    │       └── Do you want zero infrastructure management?
    │           ├── Yes → Serverless containers (Fargate, Cloud Run)
    │           └── No  → Managed containers (ECS on EC2, ACA)
```

---

## Key Takeaways

- Match compute model to workload characteristics: VMs for control, containers for portability, FaaS for event-driven
- Auto-scaling works best when you combine multiple signals (CPU, latency, queue depth) and scale out fast, in slow
- Spot instances provide 60–90% savings but require interruption-tolerant architecture: diversified fleets, checkpointing, and queue-based workers
- Right-size quarterly by analysing actual CPU and memory utilisation against instance capacity
- Default to ARM-based instances unless you have an explicit x86 requirement — the price/performance advantage is significant
- For ML workloads, separate training (spot-friendly, batch) from inference (latency-sensitive, auto-scaled)
