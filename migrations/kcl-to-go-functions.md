# KCL Compositions to Go Functions

## Overview

Replaced all `function-kcl` based Crossplane compositions with custom Go composition functions using the [Crossplane Function SDK](https://github.com/crossplane/function-sdk-go).

## Why

The `function-kcl` pod was consuming **~2.4 CPU cores** and **~480 MiB memory** to reconcile ~300 XRs. Prometheus alerts started firing for CPU throttling — it was the single biggest resource consumer in the cluster.

## What Changed

| Before | After |
|--------|-------|
| `function-kcl` + `function-auto-ready` (2 pipeline steps) | Single custom Go function (1 pipeline step) |
| KCL inline source evaluated at runtime | Compiled Go binary, statically linked |
| ~2.4 CPU cores | ~0.003 CPU cores (**800x reduction**) |
| ~480 MiB memory | ~15.5 MiB memory (**31x reduction**) |

## Compositions Migrated

- **Harbor replications** — `function-harbor-replication` (~500 lines of Go) ✅
- **Wapp** (Kubernetes deployment abstraction) — `function-wapp` (~8,500 lines of Go) ✅
- **Wdb** (CNPG PostgreSQL abstraction) — in progress
- **Wsecret** (Infisical secrets abstraction) — in progress

Completed functions include unit tests, golden file render tests, and E2E Chainsaw tests.

## Full Write-Up

For the complete story — including screenshots, tradeoffs, and when KCL still makes sense:

[From KCL to Custom Go Functions: An 800x CPU Reduction]({{< ref "crossplane/kcl-to-go-functions" >}})

