# fj-queue

<img src="/images/fj-queue.png" class="hero-image" alt="fj-queue terminal dashboard showing runner inventory, per-pod resource usage, per-repo backlog, and the waiting job queue" style="max-width: 700px; width: 100%; height: auto;" />

{{< callout type="info" >}}
**TL;DR.** `fj-queue` is a small read-only tool that polls the Forgejo admin Actions API and shows, in one view, which runners are online, what's running versus waiting, the per-repo backlog, and the FIFO-approximate queue order. Two faces from one aggregation layer: a live terminal dashboard for me, stable JSON for my agents. `brew install fj-queue`.
{{< /callout >}}

## The problem

Forgejo's web UI doesn't aggregate Actions across repos. When CI feels backed up I want one screen that answers: are the runners online, how many jobs are running versus queued, which repo is hogging the queue, and is anything stuck because no runner matches its labels? `tea` and the raw API expose fragments; nothing stitches them together.

## What it does

It hits the admin Actions API and renders runner inventory and state, running-versus-waiting totals, a FIFO-approximate queue order, per-repo backlog, and unschedulable-job warnings (a job whose `runs_on` no online runner can satisfy).

One aggregation layer, two output modes:

- **Humans**: a live, in-place refreshing terminal dashboard (Rich).
- **Agents**: a byte-deterministic JSON document (`--format json`) backed by a committed JSON Schema, so a script can branch on saturation, a repo's queue position, or a `blocked_reason`.

It is strictly read-only. No cancel, rerun, or delete, so it's safe to point at a busy production instance.

## Install

```bash
brew tap vtmocanu/tap
brew trust vtmocanu/tap    # Homebrew 6.0+ requires trusting third-party taps
brew install fj-queue
fj-queue --host git.example.com
```

Needs an admin-scoped API token via `--token` or `$FORGEJO_TOKEN`.

Source and docs live in the repo: [github.com/vtmocanu/fj-queue](https://github.com/vtmocanu/fj-queue).

