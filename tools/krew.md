# Krew — Homebrew for kubectl

<img src="/images/krew.png" alt="Krew" class="hero-image" style="max-width: 700px; width: 100%; height: auto;" />

[Krew](https://krew.sigs.k8s.io/) is essentially Homebrew for kubectl — a plugin manager that lets you discover, install, and manage kubectl plugins from a curated index of 300+ community tools. Instead of hunting down GitHub repos and wrangling binaries yourself, it's just:

```bash
kubectl krew install <plugin-name>
```

One command, and your `kubectl` gains new superpowers.

## Why Bother?

Vanilla `kubectl` is powerful, but it has gaps. Want to see PV disk usage? Decode a secret without the base64 dance? Watch pods with color-coded statuses? You're looking at multi-step pipelines of `kubectl get -o json | jq | base64 -d | ...`.

Krew plugins collapse all that into single commands. After years of running a homelab cluster with 90+ apps, these are the ones that stuck.

## My Top Picks

### [klock](https://github.com/applejag/kubectl-klock) — The Watch That Actually Works

I use this daily. It replaces `kubectl get --watch` with a live-updating table that has color-coded statuses, auto-refreshing age columns, and deleted resources that fade out instead of vanishing. Once you try it, the default `--watch` feels broken.

```bash
kubectl klock pods -A
kubectl klock deployments -n prod
kubectl klock nodes
```

### [stern](https://github.com/stern/stern) — Multi-Pod Log Tailing

Another daily driver. Tails logs from multiple pods and containers simultaneously, with color-coded output per pod. The query is a regex, so `stern api` catches `api-server-abc123` and `api-gateway-def456` at once. Pods that die get removed, new ones get picked up automatically. Beats juggling multiple `kubectl logs -f` terminals.

```bash
stern api-server                    # tail all pods matching "api-server"
stern -n prod .                     # tail everything in a namespace
stern deploy/my-app -c app          # specific deployment, specific container
stern --since 5m my-service         # last 5 minutes
```

### [neat](https://github.com/itaysk/kubectl-neat) — Clean YAML, Finally

Strips all the clutter from `kubectl get -o yaml` output — managed fields, default values, status blocks, system metadata, service account token volumes. What's left is just the manifest you actually care about. Essential when you need to grab a resource definition to reuse or debug.

```bash
kubectl get pod mypod -o yaml | kubectl neat
kubectl neat get -- deploy my-app -o yaml
```

I have it aliased as `kneat` which also pipes through `yq` for syntax-highlighted YAML:

```bash
alias kneat="kubectl neat | yq"

# Usage: kubectl get deploy my-app -o yaml | kneat
```

### [cnpg](https://cloudnative-pg.io/documentation/) — CloudNativePG's Right Hand

The official kubectl plugin for [CloudNativePG](https://cloudnative-pg.io/). I run all my PostgreSQL on CNPG, so this gets heavy use. Cluster status, triggering backups, launching `psql` sessions, promoting replicas, tailing aggregated logs — it's the CLI control plane for your Postgres clusters.

```bash
kubectl cnpg status my-cluster
kubectl cnpg psql my-cluster
kubectl cnpg backup my-cluster
kubectl cnpg logs cluster my-cluster -f
```

### [view-secret](https://github.com/elsesiy/kubectl-view-secret) — No More Base64 Gymnastics

Decodes and displays Kubernetes secret values without the `kubectl get secret -o jsonpath | base64 -d` chain. It handles all secret types — Opaque, TLS, Docker config, Helm secrets (double base64 + gzip), service account tokens. Quick, read-only, and safe.

```bash
kubectl view-secret my-secret           # list keys, pick one
kubectl view-secret my-secret my-key    # decode specific key
kubectl view-secret my-secret -a        # decode everything
```

### [modify-secret](https://github.com/rajatjindal/kubectl-modify-secret) — When You Need to Edit, Not Just View

Most of my secrets go through [Infisical](/infrastructure/the-stack/) and never need manual touching. But for the odd one-off secret that lives outside the secret manager, this is a lifesaver. It decodes the secret, opens your `$EDITOR`, and re-encodes + applies on save.

```bash
kubectl modify-secret my-secret -n kube-system
```

### [df-pv](https://github.com/yashbhutwala/kubectl-df-pv) — Disk Usage for Persistent Volumes

Unix `df` but for PVs. Shows used/available/capacity with color-coded output by severity. Simple and essential — you don't want to find out a PV is full from a crash.

```bash
kubectl df-pv
kubectl df-pv -n databases
```

### [images](https://github.com/chenjiandongx/kubectl-images) — What's Actually Running?

Lists all container images running in your cluster. Great for auditing image versions, spotting unvetted images, or just getting a quick inventory.

```bash
kubectl images -A          # all namespaces
kubectl images -n prod     # specific namespace
```

### [view-cert](https://github.com/lmolas/kubectl-view-cert) — TLS Certificate Inspector

Parses TLS secrets and shows human-readable certificate details — issuer, subject, validity dates, serial number. The `--days` flag for finding certs expiring soon is a nice companion to cert-manager.

```bash
kubectl view-cert -A                    # all TLS secrets
kubectl view-cert -n ingress --days 30  # expiring within 30 days
```

### [viewnode](https://github.com/NTTDATA-DACH/viewnode) — Node Draining Companion

Shows a hierarchical tree of nodes with their pods and containers. I mainly pull this out when draining a node — watching what's still running and waiting for eviction in real-time.

```bash
kubectl viewnode --show-containers --show-metrics
kubectl viewnode --node-filter worker-1
```

### [cwide](https://github.com/reborn1867/kubectl-cwide) — Custom Wide Output

`kubectl get -o wide` is often useless for CRDs. cwide lets you define persistent Go templates per resource kind so you get the columns that actually matter. I have custom templates for CNPG clusters that show replication status, backup info, and instance counts at a glance.

```bash
kubectl cwide get cluster.postgresql.cnpg.io -A
kubectl cwide get helmrelease -A
```

### [radar](https://github.com/skyhook-io/radar) — Kubernetes UI That Runs Locally

I wrote a separate page about this one because it deserves it: [Radar — A Kubernetes UI I Actually Like](/tools/radar/).

## Installing Krew

Full instructions at the [official install guide](https://krew.sigs.k8s.io/docs/user-guide/setup/install/). Quick version for macOS / Linux (bash/zsh):

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Then add to your `.bashrc` or `.zshrc`:

```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

After that, it's just `kubectl krew install <plugin>` for anything on the [krew plugin index](https://krew.sigs.k8s.io/plugins/).

## Links

- [Krew homepage](https://krew.sigs.k8s.io/)
- [Plugin index](https://krew.sigs.k8s.io/plugins/) — browse all 300+ plugins
- [GitHub](https://github.com/kubernetes-sigs/krew)
- [User guide](https://krew.sigs.k8s.io/docs/user-guide/quickstart/)

