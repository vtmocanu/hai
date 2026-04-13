# From s3bkp to VolSync

<img src="/images/s3bkp-to-volsync.png" alt="Transition from s3bkp (cluttered, retiring soon) to VolSync (clean, automated) with temporary worker robots and blue/green S3 buckets" style="max-width: 700px; width: 100%; height: auto;" />

{{< tabs >}}

{{< tab name="Overview" icon="book-open" >}}

## I Knew About VolSync (and Dismissed It)

When I first built [s3bkp](/custom-tools/s3bkp/), I was aware VolSync existed. I looked at it briefly, decided it wasn't mature enough for what I needed, and moved on. My custom tool worked, backed up PVCs to S3, restored them during blue/green cluster migrations, and I had full control over every aspect. Why fix what isn't broken?

That reasoning held for over a year.

## The Video That Changed My Mind

Then I watched Mircea Anton's video: [How I Backup My Kubernetes Cluster the GitOps way (Volsync)](https://www.youtube.com/watch?v=wM8skoa6W4c).

Mircea's setup looked familiar. Too familiar. He was using VolSync with Flux Kustomize Components, per-app `postBuild` variables, S3 storage with restic, automatic restore on first PVC provision via `dataSourceRef`. It was essentially the same architecture I had built with s3bkp: declarative, GitOps-native, restore-as-code. But without the 2,400 lines of bash. Without the Kyverno injection policies. Without the custom container image I had to maintain, test, and update.

That was the moment it clicked. I hadn't just built a backup tool. I had rebuilt VolSync from scratch, with worse consistency guarantees (s3bkp backs up live data; VolSync takes a VolumeSnapshot first) and a permanent resource overhead (24/7 sidecar vs. temporary mover Jobs).

## Why Not Just Use Velero?

This was the first question I asked myself. [Velero](https://velero.io/) is the standard Kubernetes backup tool, and I do use it for disaster recovery. But Velero's restore model is fundamentally imperative: you run `velero restore create`, it restores an entire namespace (or a filtered subset), and then you deploy your app on top. That works for disaster recovery, but it doesn't work for blue/green cluster migrations where I need restore-as-code.

What I needed was: commit an app to git, Flux deploys it on the new cluster, and the data is automatically restored from the old cluster's backup before the app starts. No manual `velero restore` command. No runbook. No human in the loop. Just deploy and go.

Velero can't do this because:

- **No `dataSourceRef` integration.** Velero doesn't participate in Kubernetes volume populators. You can't point a PVC at a Velero backup and have it auto-populate on first provision.
- **Restores are one-shot, not desired state.** Velero does have a `Restore` CRD, and you can technically apply it declaratively. But a Restore CR is inherently one-shot and immutable: once it transitions to `Completed`, it's done. You can't commit it to git as persistent desired state the way you would a HelmRelease. If Flux re-applies it, Velero won't re-run it. There's no "keep this PVC restored from this backup" primitive.

VolSync solves both. The `ReplicationDestination` CRD is the restore intent committed to git. The PVC's `dataSourceRef` is the declarative link. Delete the PVC, Flux recreates it, VolSync repopulates it from the latest backup. That's restore-as-code.

## What VolSync Does Better

The core difference is architecture. s3bkp runs as a Kyverno-injected sidecar that lives inside every backed-up pod, permanently consuming resources. VolSync is an operator: you declare a `ReplicationSource` CRD, and it spawns a temporary mover Job on schedule, takes a VolumeSnapshot for consistency, backs up to S3, and then the Job terminates. No permanent sidecar. No pod-level coupling.

For restores, VolSync uses Kubernetes-native `dataSourceRef` volume populators. You create a PVC that points at a `ReplicationDestination`, and Kubernetes populates the volume from the latest backup before the PVC is even available to mount. The app literally cannot start until the restore is complete. With s3bkp, I had to carefully manage restore timing in the init container to avoid race conditions where the app would overwrite restored data.

| Aspect | s3bkp | VolSync |
|--------|-------|---------|
| Architecture | Kyverno-injected sidecar | CRD-based operator |
| Backup consistency | Live filesystem (app is writing) | VolumeSnapshot first |
| Resource overhead | Permanent sidecar per pod | Temporary Job, then gone |
| Restore mechanism | Init container (timing-sensitive) | Volume populator (atomic) |
| Configuration | Pod labels + annotations | CRD resources |
| Maintenance | ~2,400 lines of bash | Upstream Helm chart |
| Monitoring | Custom 25+ metrics | Native operator metrics |

## Evaluating the Migration

Before committing, I ran a proof of concept with a single app. The evaluation checklist was straightforward:

- Can VolSync back up a PVC to my existing S3 storage (RustFS)?
- Can it restore on a fresh PVC via `dataSourceRef`?
- Can it restore cross-cluster (blue cluster reading from green's bucket)?
- Does it work with my existing Flux + Kustomize patterns?
- Can I monitor it with Prometheus?

Every box checked. The POC app was fully migrated in a single session. The Flux Kustomize Component pattern from Mircea's setup fit naturally into my existing repo structure.

## The Migration

With the POC validated, I worked through the remaining apps systematically. Each migration follows the same two-phase pattern:

**Phase 1**: Deploy VolSync backup alongside s3bkp (both run in parallel). Wait for the first VolSync backup to land in S3. Copy the backup to the other cluster's bucket for cross-cluster restore readiness.

**Phase 2**: Remove s3bkp labels, scale the app down, refresh the VolSync restore snapshot, delete the old PVC, and let Flux recreate it with `dataSourceRef`. The volume populator restores data before the app starts. Verify, done.

The two-phase approach means there's always a fallback. If VolSync's backup fails in Phase 1, s3bkp is still running. Only after verifying VolSync works do I cut over in Phase 2.

## Lessons Learned Along the Way

The migration wasn't entirely smooth. A few things I learned:

- **VolumeSnapshot before backup is not just a nice-to-have.** s3bkp backed up live data, which meant occasional inconsistencies with apps that write frequently. VolSync's snapshot-first approach eliminates this entire class of problems.
- **Restore timing matters more than you think.** With s3bkp, the init container could race with the app. VolSync's `dataSourceRef` populator runs before the PVC is even mountable, making restore atomic by design.
- **Kyverno mutation persistence is surprising.** Removing the s3bkp label from a pod template doesn't make Kyverno remove its previously injected containers. Crossplane and Helm's SSA both preserve fields they don't explicitly manage. The fix: delete the Deployment and let the controller recreate it clean.
- **Multi-PVC apps need multiple Flux Kustomizations.** Kustomize components can't be included twice in the same Kustomization (upstream limitation). For apps with multiple volumes, each PVC gets its own Flux Kustomization CR pointing at the same shared component.

## Current Status

Most apps are migrated. The remaining few are being worked through. Once complete, s3bkp's Kyverno policies will be archived and the repository decommissioned.

I don't regret building s3bkp. It taught me the real complexity behind backup tooling and served reliably for over a year. But maintaining a custom solution when a community-supported one exists is a cost I no longer need to pay. Sometimes the best code you write is the code you eventually delete.

{{< callout type="info" >}}
For the full story of what s3bkp was and how it worked, see [s3bkp: Backup-as-Code for Kubernetes PVCs](/custom-tools/s3bkp/).
{{< /callout >}}

{{< /tab >}}

{{< tab name="Technical Deep Dive" icon="code" >}}

## How I Use VolSync

The setup follows the pattern from Mircea's video: a shared [Flux Kustomize Component](https://fluxcd.io/flux/components/kustomize/kustomizations/#components) that any app can include. The component is DRY; per-app customization happens via Flux `postBuild.substitute` variables.

### Directory Layout

```treeview
components/
├── volsync/                         # Backup/restore resources
│   ├── kustomization.yaml           # kind: Component
│   ├── secret.yaml                  # Per-app S3 + restic credentials
│   ├── replication-source.yaml      # Scheduled backup to S3
│   └── replication-destination.yaml # Restore target for dataSourceRef
└── volsync-pvc/                     # Optional: VolSync-managed PVC
    ├── kustomization.yaml           # kind: Component
    └── pvc.yaml                     # PVC with dataSourceRef
```

Apps that support `existingClaim` use both components. Apps where Helm manages the PVC use only `volsync/` and inject `dataSourceRef` via a postRenderer.

### Wiring an App

Adding VolSync to an app takes a few lines in its Flux Kustomization. Here's Forgejo (my self-hosted git forge) as an example:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infra-forgejo
  namespace: flux-system
spec:
  dependsOn:
    - name: infra-volsync
  path: ./infrastructure/controllers/forgejo
  targetNamespace: forgejo
  components:
    - ../../../components/volsync/
    - ../../../components/volsync-pvc/
  postBuild:
    substitute:
      APP: forgejo
      VOLSYNC_PVC: gitea-shared-storage   # chart's PVC name
      VOLSYNC_CAPACITY: 10Gi
      VOLSYNC_PUID: "1000"                # rootless image
      VOLSYNC_PGID: "1000"
      VOLSYNC_SCHEDULE: "4 */6 * * *"
    substituteFrom:
      - kind: ConfigMap
        name: cluster-config      # provides: volsync_bucket, volsync_s3_url
      - kind: Secret
        name: volsync-creds       # provides: AWS_ACCESS_KEY_ID, RESTIC_PASSWORD
```

That's it. Flux substitutes the variables into the component templates, and the app gets a `ReplicationSource`, `ReplicationDestination`, two Secrets, and a PVC with `dataSourceRef`.

### The Credential Split

Each app gets two Secrets: one for backup, one for restore. They point at different S3 buckets:

```yaml
# Backup secret - writes to own-color bucket
apiVersion: v1
kind: Secret
metadata:
  name: "${APP}-volsync-src"
stringData:
  RESTIC_REPOSITORY: "s3:${volsync_s3_url}/${volsync_bucket}/${APP}"
  RESTIC_PASSWORD: "${RESTIC_PASSWORD}"
  AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
  AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
---
# Restore secret - reads from other-color bucket
apiVersion: v1
kind: Secret
metadata:
  name: "${APP}-volsync-dst"
stringData:
  RESTIC_REPOSITORY: "s3:${volsync_s3_url}/${volsync_restore_bucket}/${APP}"
  # ... same credentials
```

The `cluster-config` ConfigMap provides the bucket names per cluster color:

```yaml
# Blue cluster
volsync_bucket: volsync-blue           # own backups
volsync_restore_bucket: volsync-green  # restores from green
volsync_s3_url: "https://r3.example.com"
```

This means deploying an app on the blue cluster automatically backs up to `volsync-blue` and restores from `volsync-green`. No per-app configuration needed for cross-cluster restore.

### The ReplicationSource (Backup)

The backup runs on a cron schedule. Each app gets a unique minute offset to stagger S3 access:

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: "${APP}"
spec:
  sourcePVC: "${VOLSYNC_PVC:=${APP}}"
  trigger:
    schedule: "${VOLSYNC_SCHEDULE:=0 */6 * * *}"
  restic:
    copyMethod: Snapshot              # VolumeSnapshot first, then backup
    repository: "${APP}-volsync-src"
    volumeSnapshotClassName: ceph-prx-vsc
    cacheCapacity: 2Gi
    moverSecurityContext:
      runAsUser: ${VOLSYNC_PUID:=1000}
      runAsGroup: ${VOLSYNC_PGID:=1000}
      fsGroup: ${VOLSYNC_PGID:=1000}
    retain:
      hourly: 24
      daily: 7
```

The key is `copyMethod: Snapshot`. VolSync creates a VolumeSnapshot before the mover Job starts, so the backup reads from a frozen point-in-time copy while the app keeps running.

### The ReplicationDestination (Restore)

The restore side uses `trigger.manual: restore-once` and the `IfNotPresent` SSA label, which tells Flux to never overwrite this resource after creation:

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: "${APP}-dst"
  labels:
    kustomize.toolkit.fluxcd.io/ssa: IfNotPresent
spec:
  trigger:
    manual: restore-once
  restic:
    repository: "${APP}-volsync-dst"
    copyMethod: Snapshot
    capacity: "${VOLSYNC_CAPACITY:=5Gi}"
    enableFileDeletion: true
```

On first creation, the destination mover runs once, pulls the latest Restic snapshot, creates a VolumeSnapshot, and sets `status.latestImage` to that snapshot. From then on, it's idle.

### The PVC (Auto-Restore)

This is where the magic happens. The PVC's `dataSourceRef` points to the ReplicationDestination:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "${VOLSYNC_PVC:=${APP}}"
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
spec:
  storageClassName: ceph-prx
  accessModes: ["ReadWriteOnce"]
  dataSourceRef:
    kind: ReplicationDestination
    apiGroup: volsync.backube
    name: "${APP}-dst"
  resources:
    requests:
      storage: "${VOLSYNC_CAPACITY:=5Gi}"
```

When this PVC is first provisioned, Kubernetes sees the `dataSourceRef`, asks VolSync to populate the volume from the latest Restic snapshot, and the app cannot mount the PVC until the restore completes. Atomic, race-free, declarative.

The `prune: disabled` annotation is a safety net. If someone removes the app from the Flux Kustomization, the PVC (and its data) won't be garbage-collected.

### Manual Restore Workflow

When I need to restore an app (data corruption, rollback, or cluster migration), the workflow is:

```bash
# Suspend Flux so it doesn't fight us
flux suspend kustomization app-myapp

# Stop the app
kubectl scale deployment/myapp -n myapp --replicas=0

# Delete the PVC (Flux will recreate it with dataSourceRef)
kubectl delete pvc myapp -n myapp

# Resume Flux - PVC provisions from latest backup, app starts with restored data
flux resume kustomization app-myapp
```

The `dataSourceRef` populator runs before the PVC is mountable. The app cannot start until the restore is complete. No race conditions, no init container timing issues.

### Schedule Staggering

With 10+ apps backing up every 6 hours, hitting S3 simultaneously causes spikes. Each app uses a unique minute offset:

| Minute | App |
|--------|-----|
| 0 | pulse |
| 3 | certmate-data |
| 4 | forgejo |
| 5 | freshrss-data |
| 6 | nextcloud |
| 12 | vaultwarden |

A Taskfile command (`task volsync:schedules`) lists all schedules and detects conflicts.

{{< /tab >}}

{{< /tabs >}}

