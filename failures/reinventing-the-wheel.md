# Reinventing the Backup Wheel

## The Shortcut I Didn't Take

When I needed automated PVC backup and restore for my blue/green cluster migrations, I looked at Velero, decided its imperative restore model didn't fit my "everything-as-code" approach, and jumped straight into building my own tool. I didn't dig much deeper than that. Surely, if Velero couldn't do it, I'd have to build it myself.

So I built [s3bkp](/custom-tools/s3bkp/) - a Kyverno-injected sidecar that backs up PVCs to S3 and restores them automatically on pod startup. It grew to ~2,400 lines of bash, supported four backup engines, exposed 25+ Prometheus metrics, and reliably served through multiple cluster migrations over a year.

The problem? [VolSync](https://volsync.readthedocs.io/) existed. It's a CNCF-adjacent Kubernetes operator that does the same thing: backs up PVCs to S3 with restic, restores via native volume populators, runs as temporary Jobs instead of permanent sidecars. It even does things better: VolumeSnapshots before backup (consistent point-in-time copies instead of backing up live data), no custom code to maintain, CRD-based state management.

I found VolSync eventually, and migrated to it. But I could have found it a year earlier if I'd spent another afternoon researching before writing the first line of bash.

## What I Got Wrong

**I stopped searching too early.** Velero was the obvious tool, it didn't fit my needs, so I assumed there was a gap. I didn't look for less mainstream alternatives. VolSync was out there, growing steadily, and would have saved me weeks of development and ongoing maintenance.

## What I Got Right (Accidentally)

It wasn't a total failure. Building s3bkp taught me things I wouldn't have learned by just deploying someone else's operator:

- **Backup is harder than it looks.** File ownership during restore, S3 locking semantics, live backup consistency, multi-volume atomicity, retention policy edge cases. Every one of these bit me at some point. I now have a much deeper appreciation for what backup tools handle under the hood.
- **Kyverno as an injection framework.** The pattern of "label your pod, get automatic behavior" via Kyverno mutations was genuinely elegant. I still use this pattern for other cross-cutting concerns.
- **The restore side matters more than the backup side.** Everyone thinks about backups. The real complexity is in restores: when do they trigger, what happens if the app starts before the restore finishes, how do you handle cross-cluster credential differences. Building s3bkp forced me to think deeply about restore ergonomics, which made the VolSync migration smoother because I knew exactly what to look for.

## The Takeaway

Before building, search harder. "The obvious tool doesn't fit" is not the same as "no tool fits." Spend a day looking at alternatives before committing to weeks of development. The CNCF landscape is big, and the k8s-at-home community has probably solved your problem already.

That said, don't feel too bad about reinventing wheels. Sometimes the journey through the implementation teaches you things the documentation never will. Just make sure it's a conscious choice, not a shortcut past the research you should have done.

{{< callout type="info" >}}
The full migration story: [From s3bkp to VolSync](/migrations/s3bkp-to-volsync/).
{{< /callout >}}

