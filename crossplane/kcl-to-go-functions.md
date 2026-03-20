# From KCL to Custom Go Functions: An 800x CPU Reduction

I'm going to be honest: this one caught me completely off guard.

For months, I ran all my Crossplane compositions through [function-kcl](https://github.com/crossplane-contrib/function-kcl) — the KCL-based composition function. It was the recommended approach, it worked, and the DX was decent. Write some KCL, deploy, done.

Then I started noticing the resource usage.

## The Problem

My homelab cluster runs about **~300 Composite Resources (XRs)** — a mix of Harbor replications, web apps, databases, and secrets. Every one of them reconciled through a single `function-kcl` pod. And that pod was _hungry_.

I didn't discover this gradually. One day I started getting hammered by Prometheus alerts — CPU throttling, resource quota warnings, the works. The KCL function pod was eating CPU like nothing else in the cluster. That's what finally made me dig into the Grafana dashboards and face the numbers:

Here's what the KCL function looked like under normal operation:

### KCL Function — CPU Usage

![KCL function-kcl CPU usage — hovering around 2.37 cores with spikes to 3+](/images/kcl-cpu-usage.png)

**~2.37 CPU cores** sustained. With spikes above 3 cores. For a composition function.

### KCL Function — Memory Usage

![KCL function-kcl memory usage — ~479 MiB working set, requests at 700 MiB, limit 1 GiB](/images/kcl-memory-usage.png)

**~479 MiB** working set, with memory requests set to 700 MiB and a 1 GiB limit. The memory would regularly spike near the limit, and I had already bumped the limits once to avoid OOMKills.

This was one of the biggest resource consumers in the entire cluster. A single function pod was using more CPU than most of my actual workloads combined.

## What I Tried First

Before rewriting anything, I tried the obvious optimizations:

- **Increased reconciliation intervals** — helped reduce frequency, but the per-reconciliation cost was still high
- **Enabled `--enable-realtime-compositions`** on Crossplane core — eliminated polling overhead (this alone [cut reconciles by 66%](/infrastructure/reduce-apiserver-load/))
- **Reduced debug logging** on providers

These helped with overall cluster load, but the KCL function itself remained the elephant in the room. Each reconciliation still had to spin up the KCL runtime, evaluate the composition logic, and return the result. With ~300 XRs, that adds up fast.

## The Experiment: Custom Go Functions

I'd been reading about writing [custom composition functions in Go](https://docs.crossplane.io/latest/guides/write-a-composition-function-in-go/) and figured — why not try it? The worst case would be wasted time. The best case... well.

The idea is simple: instead of writing composition logic in KCL (a DSL that runs inside a runtime), you write it directly in Go using the [Crossplane Function SDK](https://github.com/crossplane/function-sdk-go). Your function compiles to a single static binary, runs as a gRPC server, and Crossplane calls it during reconciliation.

### What I Built

I started with **Harbor replications** — my most numerous XR type. The KCL composition had a two-step pipeline:

1. Step 1: KCL evaluates the XR spec and produces Harbor Project, Replication, and RetentionPolicy resources
2. Step 2: `function-auto-ready` checks readiness

I replaced both steps with a single Go function: `function-harbor-replication`. One pipeline step, one binary, zero KCL dependency.

```yaml
# Before: two KCL pipeline steps
pipeline:
  - step: compose-harbor-resources
    functionRef:
      name: function-kcl
    input:
      apiVersion: krm.kcl.dev/v1alpha1
      kind: KCLRun
      spec:
        source: |
          # ... 200+ lines of KCL ...
  - step: auto-ready
    functionRef:
      name: function-auto-ready

# After: single Go function
pipeline:
  - step: create-harbor-resources
    functionRef:
      name: function-harbor-replication
```

The Go function does everything in `RunFunction()`:

1. Extracts configuration from the XR spec (with defaults matching the old KCL behavior exactly)
2. Conditionally builds Project, Replication, and RetentionPolicy resources
3. Handles readiness detection inline (no separate function needed)
4. Extracts status fields back to the XR

Then I did the same for **Wapp** (my Kubernetes deployment application abstraction) — a much larger composition that manages Deployments, Services, Ingresses, VPAs, PVCs, ConfigMaps, databases, and secrets. The `function-wapp` Go function is about 8,500 lines of Go with comprehensive test coverage.

**Wdb** (CNPG PostgreSQL abstraction) and **Wsecret** (Infisical secrets abstraction) are next in line — currently being migrated from KCL to Go as well.

## The Results

I deployed the Go functions alongside the existing KCL setup, migrated the compositions one by one, and pulled up the dashboards. I had to double-check the pod selector because I genuinely thought the metrics were broken.

### Go Function — CPU Usage

![Custom Go function CPU usage — 0.003 cores](/images/go-function-cpu-usage.png)

**0.003 CPU cores.** That's 3 millicores. The graph barely registers.

### Go Function — Memory Usage

![Custom Go function memory usage — 15.5 MiB working set](/images/go-function-memory-usage.png)

**15.5 MiB** working set. Requests at 64 MiB, limits at 128 MiB — and I'm probably being generous.

### Side by Side

| Metric | KCL Function | Go Function | Reduction |
|--------|-------------|-------------|-----------|
| **CPU** | ~2.37 cores | ~0.003 cores | **~800x** |
| **Memory** | ~479 MiB | ~15.5 MiB | **~31x** |
| **Pipeline steps** | 2 (KCL + auto-ready) | 1 | 50% fewer |
| **Memory limit** | 1 GiB | 128 MiB | 8x lower |

That's not a typo. **800 times less CPU.** The Go function handling the same ~300 XRs uses less CPU than what most people allocate to a single sidecar container.

## Why Is The Difference So Dramatic?

A few factors compound here:

1. **KCL runtime overhead** — `function-kcl` embeds the KCL runtime (Rust-based, with Go bindings). Every reconciliation has to initialize the KCL evaluator, parse the inline KCL source, evaluate it, and return the result. This is [a known issue](https://github.com/crossplane-contrib/function-kcl/issues/273) — KCL-embedded functions have been reported to use over 1 GiB of memory under load.

2. **Compiled vs interpreted** — A custom Go function is a statically compiled binary. There's no runtime to initialize, no source to parse. The function receives a protobuf request, runs compiled Go code, and returns a protobuf response. It's about as efficient as you can get.

3. **Single shared function** — All ~300 XRs using the same composition call the same function pod. With KCL, each call pays the interpretation overhead. With Go, each call is just a function invocation in an already-running binary.

4. **Distroless image** — The Go function runs on `distroless/static-debian12:nonroot` — roughly a 20 MB image with no shell, no libc, nothing extra. Minimal attack surface, minimal memory footprint.

## The Tradeoffs

I won't pretend this was all upside. Moving to custom Go functions comes with real costs:

**Development speed is slower.** KCL lets you iterate quickly — change the inline source, deploy, done. With Go, you need to write code, compile, build a container image, push it to a registry, and update the function package. The feedback loop is longer. I've automated most of this with a [CI pipeline that versions compositions and functions together]({{< ref "decisions/crossplane-composition-versioning" >}}), but it's still more moving parts than KCL.

**More code to maintain.** The Harbor function is ~500 lines of Go. The Wapp function is ~8,500 lines. That's real code that needs tests, reviews, and maintenance. KCL compositions were more concise (though arguably harder to test).

**Testing is better, but requires more setup.** On the flip side, Go functions are _much_ more testable. I have unit tests, golden file render tests, and full E2E tests with Chainsaw running in Kind clusters. With KCL, testing was limited to `crossplane render` spot checks.

**You need Go knowledge.** KCL was designed to be approachable. Go functions require knowing Go, the Crossplane Function SDK, and Kubernetes resource types.

## Is This For Everyone?

Honestly, probably not. If you have a handful of XRs, `function-kcl` is fine. The resource overhead only becomes painful at scale.

But if you're running hundreds of XRs and wondering why your cluster resources are disappearing — look at your composition function pods. You might be surprised.

My rule of thumb now:

- **Prototyping or < 50 XRs** — KCL is great for fast iteration
- **Production with 100+ XRs** — seriously consider custom Go functions
- **Resource-constrained environments** (like a homelab) — Go functions are a no-brainer

## What Changed In The Cluster

After migrating all compositions to Go functions, the impact on the cluster was immediate:

- **Freed up ~2.4 CPU cores** — that's significant in a 3-node homelab cluster
- **Reclaimed ~450 MiB of memory** — per function pod
- **Reconciliation became faster** — compiled Go vs interpreted KCL means each reconciliation completes quicker
- **Resource limits dropped dramatically** — from 1 GiB to 128 MiB memory limits, freeing scheduling headroom

This was part of a broader optimization effort that also included [enabling realtime compositions and tuning reconciliation intervals](/infrastructure/reduce-apiserver-load/), but the function migration was by far the biggest single improvement.

## "Can I See The Code?"

My functions are heavily opinionated — they're tailored to my homelab stack (Harbor, CNPG, Infisical, Traefik) and wouldn't make much sense as generic open-source projects. But if you're curious about the implementation or want to compare notes on writing your own, feel free to ping me on [LinkedIn](https://www.linkedin.com/in/vtmocanu). Happy to share and chat about it.

## The Bottom Line

I learned this the hard way: **composition function choice matters at scale.** KCL is a fantastic tool for getting started with Crossplane compositions — the DX is great and the learning curve is gentle. But when you're running hundreds of XRs in a resource-constrained environment, the runtime overhead of an interpreted DSL adds up in ways you might not expect.

Switching to custom Go functions felt like removing a weight I didn't know I was carrying. The cluster is quieter, the nodes have headroom again, and I sleep better knowing a single function pod isn't one spike away from an OOMKill.

If you're hitting resource walls with Crossplane and haven't looked at your function pods yet — start there. The numbers might surprise you as much as they surprised me.

