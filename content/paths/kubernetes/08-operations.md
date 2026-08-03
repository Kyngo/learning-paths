---
title: "Operations"
weight: 8
---

# Operations

Running Kubernetes in production requires proficiency in debugging, monitoring, logging, upgrades, and disaster recovery. This section covers day-to-day operational tasks and the tooling that supports them.

---

## Essential kubectl Commands

| Task | Command |
|------|---------|
| List all pods (all namespaces) | `kubectl get pods -A` |
| Describe a pod (events + config) | `kubectl describe pod <name> -n <ns>` |
| Get pod logs | `kubectl logs <pod> -n <ns>` |
| Follow logs in real time | `kubectl logs -f <pod> -n <ns>` |
| Logs from a previous crash | `kubectl logs <pod> --previous` |
| Exec into a container | `kubectl exec -it <pod> -- /bin/sh` |
| Port-forward for local access | `kubectl port-forward svc/<name> 8080:80` |
| Get events sorted by time | `kubectl get events --sort-by=.metadata.creationTimestamp` |
| Get resource YAML | `kubectl get pod <name> -o yaml` |
| Apply a manifest | `kubectl apply -f manifest.yaml` |
| Dry-run a change | `kubectl apply -f manifest.yaml --dry-run=server` |
| Scale a deployment | `kubectl scale deployment <name> --replicas=5` |
| Rollout status | `kubectl rollout status deployment/<name>` |
| Rollback a deployment | `kubectl rollout undo deployment/<name>` |
| Top nodes (resource usage) | `kubectl top nodes` |
| Top pods | `kubectl top pods -A --sort-by=memory` |

---

## Debugging Workflow

When a pod is unhealthy, follow this escalation path:

```text
1. kubectl get pods -n <ns>           → Check STATUS column
2. kubectl describe pod <name>        → Read Events section
3. kubectl logs <pod> --previous      → See crash output
4. kubectl exec -it <pod> -- sh       → Inspect inside the container
5. kubectl get events --sort-by=time  → Cluster-wide issues
```

### Common Pod Failure States

| Status | Meaning | Likely Cause |
|--------|---------|-------------|
| `Pending` | Not scheduled | Insufficient resources, node affinity mismatch |
| `ImagePullBackOff` | Can't pull image | Wrong image name, missing pull secret |
| `CrashLoopBackOff` | Repeated crashes | App error, bad config, missing dependency |
| `OOMKilled` | Out of memory | Memory limit too low |
| `Evicted` | Node under pressure | Node disk or memory exhausted |
| `Init:Error` | Init container failed | Dependency not ready |

### Debugging DNS

```bash
# Run a debug pod with networking tools
kubectl run debug --image=busybox:1.36 --rm -it -- sh

# Inside the pod:
nslookup my-service.my-namespace.svc.cluster.local
wget -qO- http://my-service:8080/health
```

---

## Monitoring

### Metrics Server

Provides `kubectl top` functionality:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top nodes
kubectl top pods -A
```

### Prometheus + Grafana Stack

```text
┌─────────────┐     ┌────────────┐     ┌─────────┐
│ Exporters   │────▶│ Prometheus │────▶│ Grafana │
│ (node, kube │     │ (scrape +  │     │ (dashb- │
│  state, app)│     │  store)    │     │  oards) │
└─────────────┘     └────────────┘     └─────────┘
                          │
                          ▼
                    ┌────────────┐
                    │Alertmanager│
                    └────────────┘
```

Key components:
- **kube-state-metrics** — object state (deployment replicas, pod phase)
- **node-exporter** — node-level CPU, memory, disk, network
- **Alertmanager** — routes alerts to Slack, PagerDuty, email
- **ServiceMonitor** CRD — tells Prometheus which services to scrape

Install via Helm:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

---

## Logging

### Architecture Options

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| Node-level DaemonSet | Agent reads `/var/log/containers/` | Simple, no app changes | Can't enrich with app context |
| Sidecar container | Logging container per pod | Fine-grained control | More resource overhead |
| Application-direct | App ships logs to backend | Full control | Couples app to logging infra |

### Fluent Bit (Lightweight Forwarder)

Fluent Bit is the standard lightweight log forwarder for Kubernetes, deployed as a DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.1
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
```

Typical pipeline: **Fluent Bit** (collect + filter) → **Elasticsearch/OpenSearch** (store) → **Kibana** (query)

---

## Cluster Upgrades

### Upgrade Strategy

```text
1. Read release notes for the target version
2. Check deprecation warnings (kubectl get apiservices)
3. Upgrade control plane first
4. Upgrade worker nodes (rolling: cordon → drain → upgrade → uncordon)
5. Upgrade add-ons (CoreDNS, kube-proxy, CNI, CSI)
6. Run smoke tests
```

### Draining a Node

```bash
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# After upgrade:
kubectl uncordon <node-name>
```

### Version Skew Policy

| Component | Supported Skew |
|-----------|---------------|
| kubelet vs API server | N-2 minor versions behind |
| kubectl vs API server | +/- 1 minor version |
| Control plane components | Must match within same minor |

---

## Backup & Disaster Recovery

### etcd Backup

etcd holds all cluster state. Backing it up is non-negotiable:

```bash
# Snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored
```

### Velero (Application-Level Backup)

```bash
# Install
velero install --provider aws --bucket my-velero-bucket \
  --secret-file ./credentials-velero \
  --backup-location-config region=eu-central-1

# Create backup
velero backup create my-backup --include-namespaces production

# Schedule daily backups
velero schedule create daily --schedule="0 2 * * *" --ttl 168h

# Restore
velero restore create --from-backup my-backup
```

### Disaster Recovery Scenarios

| Scenario | Recovery Method | RTO |
|----------|----------------|-----|
| Single pod failure | Kubernetes self-heals (ReplicaSet) | Seconds |
| Node failure | Pods rescheduled to healthy nodes | 1-5 min |
| Control plane failure (managed) | Provider auto-recovers | Minutes |
| Control plane failure (self-hosted) | Restore etcd snapshot | 15-60 min |
| Full cluster loss | Recreate + Velero restore | 1-4 hours |

---

## Managed vs Self-Hosted Kubernetes

| Aspect | EKS (AWS) | GKE (Google) | AKS (Azure) | Self-Hosted |
|--------|-----------|-------------|-------------|-------------|
| Control plane mgmt | AWS-managed | Google-managed | Azure-managed | You manage |
| Control plane cost | ~$73/mo | Free (Standard) | Free | Your infra |
| Upgrade automation | You trigger | Auto-upgrade option | Auto-upgrade option | Fully manual |
| etcd backup | Automated | Automated | Automated | Your job |
| CNI | VPC CNI (native) | Dataplane V2 | Azure CNI | Any (Cilium, Calico) |
| IAM integration | IRSA | Workload Identity | Workload Identity | Manual OIDC |
| Node management | Managed Groups, Karpenter | Auto-Provisioning | Node Pools | You manage |
| Best for | AWS shops | GCP/ML workloads | Azure/Windows | Full control, on-prem |

### When to Choose Managed

- Team has fewer than 2 dedicated platform engineers
- You want HA control plane without effort
- Cloud-native integrations (IAM, logging, LB) outweigh portability

### When to Choose Self-Hosted

- On-premises or air-gapped environments
- Full control over control plane configuration required
- Multi-cloud with consistent tooling (Cluster API)
- Cost optimisation at very large scale

---

## Key Takeaways

1. **Debug systematically** — `get` → `describe` → `logs` → `exec` → `events`. Don't skip steps.
2. **Monitoring is not optional** — at minimum deploy metrics-server; for production use Prometheus + Grafana.
3. **Centralised logging** via DaemonSet (Fluent Bit) is the standard pattern — don't rely on `kubectl logs` in production.
4. **Upgrade carefully** — control plane first, nodes second, always respecting version skew policy.
5. **Back up etcd** on a schedule for self-hosted clusters. Use Velero for application-level backup regardless of cluster type.
6. **Choose managed Kubernetes** unless you have a strong reason not to — the operational overhead of self-hosting is substantial.
