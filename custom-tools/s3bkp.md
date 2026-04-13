# s3bkp: Backup-as-Code for Kubernetes PVCs

<img src="/images/s3bkp.png" alt="Over-engineered bash machine labeled s3bkp with Kyverno, rustic, restic, rclone engines and blue/green S3 vaults" style="max-width: 700px; width: 100%; height: auto;" />

I'm a big believer in everything-as-code. I run my Kubernetes clusters in a blue/green pattern. When it's time to upgrade, I don't patch in place. I provision a fresh cluster (the new color), migrate workloads over, verify everything works, and then decommission the old one. It's clean, reproducible, and gives me a rollback path if things go sideways.

But it creates a problem: how do you get your data onto the new cluster?

## The Problem with Velero

The obvious answer is [Velero](https://velero.io/). It can back up Kubernetes resources and PVC data, and restore them on a different cluster. I used it (and still do, for disaster recovery). But for blue/green migrations, Velero has a friction point: restores are imperative.

You run `velero restore create`, wait for it to finish, then deploy your app. That's fine for a handful of services. But with 15+ apps, each with their own PVCs, the migration becomes a manual runbook of "restore X, wait, deploy Y, verify, repeat." I wanted something declarative: deploy the app on the new cluster, and the data just appears. Restore as code. A migration that requires a human to run imperative commands in the right order doesn't qualify.

## What I Built

s3bkp is a backup/restore system that runs as a sidecar container, automatically injected into pods by [Kyverno](https://kyverno.io/) policies. The idea is simple: apps opt in with a single label, and Kyverno handles the rest.

### How Apps Opt In

A single label on the pod template triggers everything:

```yaml
podLabels:
  my.domain/s3-backup: "enabled"
podAnnotations:
  my.domain/s3-backup-volume: "config"
```

That's it. Kyverno's ClusterPolicy matches on the label and mutates the pod spec to inject:

1. An **init container** that restores data from S3 before the app starts
2. A **sidecar container** that runs scheduled backups to S3

The annotations control behavior: which volumes to back up, the backup engine, schedule, retention policy, and whether to restore from the same cluster or the other color's bucket.

### The Kyverno Injection Pattern

This was the most interesting design decision. Instead of requiring each app to include backup containers in its manifest, Kyverno injects them transparently. Three ClusterPolicies handle different concerns: one injects the init container (restore), one injects the sidecar (backup), and one generates the credentials and PodMonitor.

The policies use `preconditions` to match on the `my.domain/s3-backup: enabled` label, then `foreach` loops to iterate over the volumes listed in the annotation, dynamically adding volume mounts at `/data/<volume-name>` for each one. This means the same policy handles an app with one volume or five, without any per-app changes to the policy itself.

The result: apps have zero awareness of s3bkp. No changes to Helm charts. No extra containers in git. Kyverno adds them at admission time.

### Backup Engines

s3bkp supports multiple backup backends depending on the use case:

- **Rustic** (default): A Rust-based tool compatible with the restic format. Fast, encrypted, deduplicated snapshots with retention policies.
- **Restic**: The classic. Encrypted, deduplicated, well-proven.
- **Native**: Simple rclone sync with daily tar.gz archives. No deduplication, but easy to browse.
- **Velero**: Restore-only mode. Can read from an existing Velero/Kopia backup repository, useful for initial migration from Velero to s3bkp.

### Blue/Green Restore

The key feature for cluster migrations. Each cluster color has its own S3 bucket:

```
s3bkp-blue/    # Blue cluster's backups
  rustic/namespace/app/
s3bkp-green/   # Green cluster's backups
  rustic/namespace/app/
```

The init container knows which color it's running on and, by default, restores from the *other* color's bucket. So when I deploy an app on the new green cluster, the init container pulls data from `s3bkp-blue` automatically. The app starts with its data already in place, no manual restore step needed.

After the app is running on the new cluster, the sidecar begins backing up to `s3bkp-green`. The old bucket remains as a fallback until the migration is verified.

### Migration Freeze

For apps that need a clean final backup before the switchover, s3bkp supports a "migration freeze" mode. Setting `my.domain/s3-backup-migration-freeze: "true"` makes the init container perform one last backup, then hold the pod in an init state. The app never starts. This gives you a guaranteed-consistent snapshot to restore from on the other side.

### Monitoring

The sidecar exposes Prometheus metrics on port 9090: backup duration, data size, snapshot count, repository health, and error counters. Kyverno also generates a PodMonitor per workload, so Prometheus discovers the metrics automatically.

## The Numbers

At its peak, s3bkp consisted of:

- ~2,400 lines of bash in a single `entrypoint.sh`
- 3 Kyverno ClusterPolicies (init injection, sidecar injection, secret/monitoring generation)
- Alpine-based container with 5 backup tools baked in (rustic, restic, rclone, kopia, socat)
- 25+ custom Prometheus metrics
- PrometheusRules with critical and warning alerts
- A Grafana dashboard for backup health

It worked. For several months, it reliably backed up and restored PVCs across cluster migrations. Every blue/green transition was smooth.

## Why I Replaced It

But maintaining ~2,400 lines of bash backup tooling was a tax I didn't need to keep paying. Every edge case (multi-volume apps, non-root containers, file ownership during restore, S3 credential rotation) added more complexity to an already dense script.

Then I looked at what [VolSync](https://volsync.readthedocs.io/) had become. It's a CNCF-adjacent operator that does exactly what s3bkp does: backs up PVCs to S3 using restic, restores via Kubernetes-native volume populators, runs as temporary Jobs instead of permanent sidecars. No custom code to maintain. No Kyverno injection needed. CRD-based state instead of sidecar logs.

The critical realization: I had [reinvented the wheel](/failures/reinventing-the-wheel/). A well-maintained, community-supported wheel already existed, and it was better at consistency (VolSync takes a VolumeSnapshot before backing up, while s3bkp backs up live data) and resource efficiency (temporary mover jobs vs. permanent sidecars).

The migration is currently in progress. Most apps are already on VolSync, and s3bkp will be decommissioned once the remaining few are migrated.

{{< callout type="info" >}}
Read the full migration story: [From s3bkp to VolSync](/migrations/s3bkp-to-volsync/) covers the evaluation, the two-phase migration process, and the lessons learned along the way.
{{< /callout >}}

## Lessons Learned

1. **Kyverno as an injection framework is powerful.** The pattern of "label your pod, get automatic behavior" is incredibly clean. I'd use it again for other cross-cutting concerns.

2. **Build vs. buy isn't always obvious.** When I started s3bkp, VolSync's restic mover wasn't mature enough for my use case. By the time it was, I had a working system and inertia to overcome. Periodically re-evaluating "does my custom tool still need to exist?" is healthy.

3. **Bash doesn't scale.** For a 200-line script, bash is fine. At 2,400 lines with multiple backup engines, retry logic, metrics exposition, and signal handling, it becomes a maintenance burden. If I were rebuilding it today, it would be a small Go binary.

4. **Restore-as-code is the right model.** Whether it's s3bkp's init containers or VolSync's `dataSourceRef` volume populators, having the restore happen declaratively as part of the deploy is the right abstraction for blue/green migrations. "Deploy app, data appears" is the developer experience I wanted, and both tools deliver it.

