# Vertical Pod Autoscaler

The [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) (VPA) automatically adjusts container resource requests based on observed usage. Instead of guessing memory requirements, VPA watches actual consumption and updates requests accordingly.

## Why VPA?

Setting resource requests manually is a guessing game:

- **Too low**: Pods get OOM-killed or throttled
- **Too high**: Cluster resources are wasted, scheduling becomes harder

VPA solves this by continuously analyzing usage patterns and recommending (or automatically applying) appropriate resource values.

## How I Use It

### Memory-Only Management

I configure VPA to manage **memory requests only**, leaving CPU alone:

```yaml
resourcePolicy:
  containerPolicies:
    - containerName: myapp
      controlledResources:
        - memory           # Only memory, not CPU
      controlledValues: "RequestsOnly"  # Adjust requests, not limits
      minAllowed:
        memory: 50Mi
      maxAllowed:
        memory: 512Mi
```

**Why memory-only?**

CPU and memory behave fundamentally differently:

- **CPU is compressible**—when a pod exceeds its limit, it gets throttled but keeps running. CPU limits can actually hurt performance by preventing pods from using available CPU cycles.
- **Memory is incompressible**—once allocated, it can't be reclaimed without killing the process. OOM kills are disruptive.

By letting VPA manage memory while leaving CPU alone, you get right-sized memory requests without risking CPU throttling issues. See [Stop Using CPU Limits](https://home.robusta.dev/blog/stop-using-cpu-limits) for a deeper dive into why CPU limits are often counterproductive.

### Tighter Recommendations

By default, VPA targets the 90th percentile of observed usage, which can be overly conservative. I configure the recommender to use the 70th percentile for tighter allocations:

```yaml
vpa:
  recommender:
    extraArgs:
      target-memory-percentile: "0.7"
```

This results in requests closer to actual usage while still leaving reasonable headroom.

### Update Modes

| Mode | Behavior | When to Use |
|------|----------|-------------|
| `Auto` | Evicts pods to apply new requests | Most workloads |
| `Initial` | Only sets requests on pod creation | Stateful apps sensitive to restarts |
| `Off` | Recommendations only, no changes | Observing before enabling |

## VPA Locations

VPAs can live in two places depending on how the app is deployed:

| Location | When to Use |
|----------|-------------|
| App's own repo (`manifests/vpa.yaml`) | Standard apps—lifecycle coupled with app |
| Crossplane compositions | Apps deployed via compositions that include VPA |

The key principle is co-locating VPA with the application. When the app is removed from the cluster, its VPA goes with it.

## Handling Sidecars

For pods with injected sidecars, I disable VPA control to avoid running into issues. I have a custom s3bkp backup solution that injects sidecars via Kyverno—I dont let VPA manage those:

```yaml
containerPolicies:
  - containerName: myapp
    minAllowed:
      memory: 50Mi
    maxAllowed:
      memory: 512Mi
    controlledResources:
      - memory
  # Sidecar managed externally - VPA hands off
  - containerName: backup-sidecar
    mode: "Off"
```

## Links

- [VPA GitHub Repository](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Kubernetes Docs: Vertical Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/vertical-pod-autoscale/)

