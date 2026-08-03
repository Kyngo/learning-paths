---
title: "Containers in Production"
weight: 7
---

## Beyond `docker run`

Running a container locally is one thing. Running dozens — or thousands — of containers reliably in production requires orchestration, security hardening, observability, and disciplined image management. This section bridges the gap between local Docker usage and production-grade container infrastructure.

---

## Container Orchestration Concepts

### Why Orchestrate?

A single container on a single host has no resilience. If it crashes, nothing restarts it. If traffic spikes, nothing scales it. Orchestration solves these problems:

| Problem | What Orchestration Provides |
|---------|----------------------------|
| Container crashes | Automatic restart and health-based replacement |
| Traffic spikes | Horizontal scaling (add more replicas) |
| Host failure | Reschedule containers on healthy nodes |
| Deployment | Rolling updates with zero downtime |
| Networking | Service discovery between containers |
| Resource contention | Scheduling based on CPU/memory availability |

### Core Concepts

**Desired State Reconciliation** — You declare *what* you want (e.g., "run 3 replicas of service X"), and the orchestrator continuously works to make reality match your declaration. If a container dies, the orchestrator starts a new one without human intervention.

**Scheduling** — The orchestrator decides *which host* runs each container. It considers available CPU, memory, affinity rules (keep related containers together), anti-affinity rules (spread replicas across failure domains), and constraints (e.g., GPU-only nodes).

**Service Discovery** — Containers need to find each other. Rather than hardcoding IP addresses that change constantly, orchestrators provide DNS-based or environment-variable-based discovery. A container asks for "the database service" and gets routed to a healthy instance automatically.

**Load Balancing** — Traffic is distributed across replicas. The orchestrator tracks which instances are healthy and routes only to those that pass health checks.

### Orchestration Platforms Compared

| Platform | Best For | Complexity | Managed Options |
|----------|----------|------------|-----------------|
| Docker Compose | Development, single-host apps | Low | — |
| Docker Swarm | Small teams, simple production | Medium | — |
| Amazon ECS | AWS-native container workloads | Medium | AWS Fargate |
| Kubernetes (K8s) | Large-scale, multi-cloud, complex apps | High | EKS, GKE, AKS |
| Nomad (HashiCorp) | Mixed workloads (containers + VMs + binaries) | Medium | HCP Nomad |

---

## Container Registries

A registry is where container images are stored, versioned, and distributed — think of it as "npm/PyPI for containers."

### Push/Pull Workflow

```text
Developer Machine           Registry              Production Host
─────────────────          ─────────              ───────────────
docker build .        →    docker push        →   docker pull
  (creates image)          (uploads to registry)  (downloads image)
                           stores layers           runs container
                           tracks tags
```

```bash
# Build and tag for a registry
docker build -t registry.example.com/myapp:1.2.0 .

# Authenticate
docker login registry.example.com

# Push to registry
docker push registry.example.com/myapp:1.2.0

# Pull from registry (usually automated by orchestrator)
docker pull registry.example.com/myapp:1.2.0
```

### Registry Options

| Registry | Type | Key Feature |
|----------|------|-------------|
| Docker Hub | Public/Private | Default registry, public images |
| Amazon ECR | Private (AWS) | IAM integration, lifecycle policies |
| GitHub Container Registry | Public/Private | Tied to GitHub repos |
| Google Artifact Registry | Private (GCP) | Multi-format (Docker, Maven, npm) |
| GitLab Container Registry | Private | Built into GitLab CI/CD |
| Harbor | Self-hosted | Policy enforcement, replication |

### Tagging Strategies

Tags identify specific versions of an image. Poor tagging causes deployment chaos.

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| Semantic versioning | `myapp:1.2.3` | Clear, rollback-friendly | Manual bump required |
| Git SHA | `myapp:a1b2c3d` | Traceable to exact commit | Not human-readable |
| `latest` | `myapp:latest` | Simple | Non-deterministic, dangerous in prod |
| Date-based | `myapp:2024-01-15` | Easy chronology | Multiple builds per day collide |
| Branch + SHA | `myapp:main-a1b2c3d` | Clear source | Verbose |

**Best practice:** Use immutable tags (semver or git SHA) in production. Reserve `latest` for development only. Never deploy `latest` to production — you cannot know what version is running or roll back reliably.

### Vulnerability Scanning

Registries can scan images for known CVEs in OS packages and application dependencies:

```bash
# Scan locally with Trivy
trivy image myapp:1.2.0

# Scan with Docker Scout
docker scout cves myapp:1.2.0

# Scan with Grype
grype myapp:1.2.0
```

Most managed registries (ECR, GCR, Harbor) offer automatic scanning on push.

---

## Image Security

### Base Image Selection

Your base image is your security foundation. Every package in the base image is a potential vulnerability.

| Base Image | Size | Use Case | Security Posture |
|------------|------|----------|-----------------|
| `ubuntu:24.04` | ~77 MB | General purpose, familiar | Moderate (many packages) |
| `debian:bookworm-slim` | ~52 MB | Smaller general purpose | Better (slim removes docs/extras) |
| `alpine:3.20` | ~7 MB | Minimal, security-focused | Good (musl, fewer packages) |
| `distroless` (Google) | ~2-20 MB | Production only (no shell) | Excellent (minimal attack surface) |
| `scratch` | 0 MB | Static binaries (Go, Rust) | Best (nothing to exploit) |

**Rules of thumb:**
- Use the smallest base that meets your needs
- Prefer `-slim` variants over full images
- Use `distroless` for production when you don't need shell access for debugging
- Pin base image versions (not `latest`)

### Multi-Stage Builds (Security Perspective)

Multi-stage builds are covered in the Docker section, but their security benefit deserves emphasis:

```dockerfile
# Stage 1: Build (has compilers, dev tools — large attack surface)
FROM node:22 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production (minimal — no build tools, no source code)
FROM node:22-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/server.js"]
```

The final image contains only what's needed to *run* — no compilers, no source code, no dev dependencies. This dramatically reduces the attack surface.

### Image Signing and Trust

Image signing proves that an image was built by a trusted party and hasn't been tampered with:

| Tool | Approach | Used By |
|------|----------|---------|
| Docker Content Trust (DCT) | Notary-based signatures | Docker Hub |
| Cosign (Sigstore) | Keyless signing with transparency log | Modern standard |
| Notation | OCI-native signatures | Azure, AWS |

```bash
# Sign with Cosign (keyless, uses OIDC identity)
cosign sign registry.example.com/myapp:1.2.0

# Verify
cosign verify registry.example.com/myapp:1.2.0
```

### Scanning Tools Comparison

| Tool | Type | Best For |
|------|------|----------|
| Trivy | Open source | All-in-one (images, IaC, secrets) |
| Grype | Open source | Fast, Anchore ecosystem |
| Docker Scout | Commercial (free tier) | Docker Desktop integration |
| Snyk Container | Commercial | Developer workflow integration |
| AWS ECR scanning | Managed | Automatic on push (AWS-native) |

---

## Container Logging and Monitoring

### The stdout/stderr Convention

Containers follow a simple logging rule: **write everything to stdout and stderr.** Don't write to files inside the container.

```text
Application → stdout/stderr → Container Runtime → Log Driver → Aggregation
```

Why:
- The container runtime captures stdout/stderr automatically
- Log drivers forward logs to any backend (CloudWatch, Datadog, ELK, Loki)
- No log files fill up the container filesystem
- Logs survive container restarts (they're outside the container)
- Consistent collection regardless of application language or framework

```python
# Python — correct (logs to stdout)
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
logger.info("Request processed")

# Wrong — logs to file inside container
logging.basicConfig(filename='/var/log/app.log')  # Don't do this
```

### Log Aggregation Patterns

| Pattern | How It Works | Example |
|---------|-------------|---------|
| Sidecar | Log agent container runs alongside app container | Fluentd sidecar in same pod |
| DaemonSet | One log agent per host collects from all containers | Datadog Agent on each node |
| Log driver | Container runtime ships logs directly | AWS FireLens, Docker log drivers |
| Direct SDK | Application sends to log service | Structured JSON to stdout → CloudWatch |

### Health Checks

Health checks let the orchestrator know if a container is functioning correctly:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

### Readiness vs. Liveness Probes

| Probe Type | Question It Answers | Failure Action |
|-----------|--------------------|-|
| **Liveness** | "Is the process alive?" | Kill and restart the container |
| **Readiness** | "Can it serve traffic?" | Remove from load balancer (don't kill) |
| **Startup** | "Has it finished starting?" | Wait (don't check liveness yet) |

**Example scenarios:**
- App is deadlocked → Liveness probe fails → Container restarted
- App is loading a large cache → Readiness probe fails → No traffic routed until ready
- App takes 60s to start → Startup probe gives it time before liveness kicks in

---

## Bridge to Kubernetes

### What Kubernetes Solves

Kubernetes (K8s) is the industry-standard container orchestrator. It implements all the orchestration concepts above — scheduling, service discovery, desired state, scaling — at massive scale.

| You Need | K8s Provides |
|----------|-------------|
| "Run 5 copies of my API" | Deployments + ReplicaSets |
| "Route traffic to healthy instances" | Services + Ingress |
| "Scale based on CPU usage" | Horizontal Pod Autoscaler |
| "Roll out a new version safely" | Rolling update strategy |
| "Run a job every night at 2am" | CronJobs |
| "Store config outside the image" | ConfigMaps + Secrets |
| "Persistent storage for databases" | PersistentVolumeClaims |

### Key Abstractions (High Level)

**Pod** — The smallest deployable unit. One or more containers that share networking and storage. Usually one main container per pod.

**Service** — A stable network endpoint for a set of pods. Pods come and go, but the Service DNS name stays constant. Internal services get cluster-internal IPs; external services get load balancers.

**Deployment** — Declares the desired state for a set of pods: which image, how many replicas, resource limits, update strategy. The Deployment controller creates and manages ReplicaSets to achieve the desired state.

**Namespace** — A logical partition of a cluster. Teams or environments (dev, staging) get their own namespace for isolation.

### Kubernetes vs. Simpler Alternatives

| Factor | ECS/Fargate | Kubernetes |
|--------|-------------|------------|
| Learning curve | Lower | Steep |
| Operational overhead | Managed by AWS | You manage (unless using EKS/GKE) |
| Flexibility | AWS-only, opinionated | Any cloud, highly configurable |
| Ecosystem | AWS integrations | Massive (Helm, Istio, ArgoCD, ...) |
| Best for | AWS-native teams, simpler apps | Multi-cloud, complex microservices |
| YAML complexity | Moderate (task definitions) | High (many resource types) |

**When to choose Kubernetes:** Multi-cloud requirements, complex service mesh needs, large engineering team with platform expertise, or when the ecosystem tooling (Helm charts, operators, GitOps) solves real problems for you.

**When to avoid Kubernetes:** Small team, single cloud provider, fewer than ~10 services, or when a managed service (ECS, Cloud Run, App Runner) meets your needs with less operational burden.

---

## Key Takeaways

1. **Orchestration solves resilience, scaling, and deployment** — You declare desired state; the system reconciles reality to match
2. **Use immutable image tags in production** — Semantic versions or git SHAs, never `latest`
3. **Scan images for vulnerabilities** — Automate scanning on push to your registry; block critical CVEs from deploying
4. **Minimise your base image** — Smaller images mean fewer vulnerabilities and faster pulls; prefer slim/distroless/scratch
5. **Log to stdout/stderr** — Let the platform handle log collection and routing, not your application
6. **Implement health checks** — Liveness probes detect crashes; readiness probes prevent traffic to unhealthy instances
7. **Choose orchestration complexity to match your needs** — ECS for AWS simplicity, Kubernetes for flexibility and scale
8. **Multi-stage builds are a security tool** — They keep build dependencies out of your production image
