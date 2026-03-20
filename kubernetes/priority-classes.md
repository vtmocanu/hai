# PriorityClasses

## The Problem

When a Kubernetes node runs low on memory, the kubelet starts evicting pods. Without priority classes, eviction order is essentially random — based on resource usage patterns, not business importance. Your PostgreSQL database might get killed before some throwaway cronjob. Your ingress controller dies while a debug pod survives.

I've seen this happen: node memory pressure hits, and suddenly Harbor (my container registry) gets evicted while low-priority batch jobs keep running. Every deployment in the cluster fails because it can't pull images. Not fun.

## The Solution

[PriorityClasses](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) let you define explicit eviction order. Higher priority = survives longer. When pressure hits, Kubernetes evicts low-priority pods first, giving your critical services room to breathe.

It's native Kubernetes — no operators, no external tools. Just declare which workloads matter most.

## How It Works

- Higher numeric value = higher priority
- During resource pressure, lower-priority pods get evicted first
- Pods without a PriorityClass default to priority 0
- `preemptionPolicy` controls whether a pod can preempt (evict) others to schedule

## My Two-Tier Strategy

I use two custom priority classes on top of the Kubernetes system defaults:

| PriorityClass | Value | Use Case |
|---------------|-------|----------|
| `system-node-critical` | 2B | K8s node components (built-in) |
| `system-cluster-critical` | 2B | K8s cluster components (built-in) |
| `critical-service` | 800M | **My infrastructure services** |
| `high-priority-service` | 100M | **My databases and important apps** |
| *(no class)* | 0 | Everything else |

### Critical Service (800M)

Infrastructure the cluster depends on — if these die, everything breaks:

| App | Why Critical |
|-----|--------------|
| traefik | Ingress controller — no traffic gets in without it |
| harbor | Container registry — deployments fail without images |
| cnpg | CloudNativePG operator — manages all databases |
| infisical | Secrets management — apps can't start without secrets |
| valkey | Redis cache — session storage, rate limiting |
| forgejo | Git hosting — CI/CD stops without it |

### High Priority Service (100M)

All PostgreSQL clusters (managed by CNPG) get this priority. Databases should survive longer than the applications that depend on them — losing the app is recoverable, losing the database is painful.

## The Result

Predictable failure modes. When memory pressure hits:

1. Random workloads without priority classes evict first (priority 0)
2. Then high-priority apps (100M) if pressure continues
3. Critical infrastructure (800M) survives longest
4. System components (2B) are essentially protected

I know exactly what dies first. No more "why did Harbor go down before that test deployment?"

## Implementation

Deploy the PriorityClasses to your cluster:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-service
value: 100000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "High priority for databases and important apps"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-service
value: 800000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Critical infrastructure that should never be evicted"
```

Reference them in your workloads:

```yaml
# For critical infrastructure
traefik:
  priorityClassName: critical-service

# For CNPG-managed databases
wdb:
  cluster:
    priorityClassName: high-priority-service
```

## Links

- [Kubernetes Docs: Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)

