# The Stack

## Main Site

### Compute

3x Proxmox nodes ([Minisforum MS-A2](https://www.amazon.de/MINIS-FORUM-MS-A2-Threads-Barebone/dp/B0F61ZLRBJ?th=1))

![Proxmox Resources](/images/proxmox-resources.png)

**Raspberry Pi 4 Model B** - 8GB RAM

- Backup DNS server ([AdGuard Home](https://adguard.com/en/adguard-home/overview.html))
- [Tailscale](https://tailscale.com/) exit node

### Storage

**Synology DS920+** - 4x Seagate IronWolf Pro 8TB (RAID5)

**Synology DX517** expansion - 4x Seagate Exos 7E10 8TB (SHR)

**S3:** [Versity Gateway](https://github.com/versity/versitygw) + [RusticFS](https://rustic.cli.rs/) - ~2TB total

- Migrated from [MinIO](https://min.io/) (also tried [Garage](https://garagehq.deuxfleurs.fr/))

### Networking

- **D-Link DGS-1210-24** — 24-port managed Gigabit switch
- **3x Tenda SM108** — 8-port 2.5GbE unmanaged switches
- **3x UniFi AC LR** — Long-range WiFi access points

### Kubernetes

Deployed via [Spectro Cloud](https://www.spectrocloud.com/) (migrated from [Kubespray](https://kubespray.io/))

- [Cilium](https://cilium.io/) — CNI, network policies, LoadBalancer IPAM
- [Traefik](https://traefik.io/) — Ingress controller (public + private entrypoints)
- [cert-manager](https://cert-manager.io/) — Automated TLS certificates (Let's Encrypt)
- [Rook-Ceph](https://rook.io/) — Distributed storage for PVCs
- [VPA]({{< ref "kubernetes/vpa" >}}) — 92 policies auto-tuning resource requests
- [Reloader](https://github.com/stakater/Reloader) — auto-restart pods on ConfigMap/Secret changes
- [Descheduler](https://github.com/kubernetes-sigs/descheduler) — rebalance pods across nodes
- [k8tz](https://github.com/k8tz/k8tz) — timezone injection for pods
- [etcd-defrag](https://github.com/ahrtr/etcd-defrag) — automatic etcd maintenance
- [PriorityClasses]({{< ref "kubernetes/priority-classes" >}}): `critical-service` (800M) for infra, `high-priority-service` (100M) for apps

### GitOps

[Flux CD](https://fluxcd.io/) managing 90+ applications (migrated from [ArgoCD]({{< ref "migrations/argocd-to-flux" >}}))

### Git & CI

**[Forgejo](https://forgejo.org/)** - 230+ repos organized via organizations

- Migrated from GitLab (self-hosted)

![Forgejo Repos](/images/forgejo-repos.png)

**[Forgejo Actions](https://forgejo.org/docs/latest/user/actions/)** - 67 workflows (GitHub Actions compatible)

- Migrated from GitLab CI + GitLab CI/CD Catalog

**[Crossplane](https://www.crossplane.io/) CI** - Compositions with full CI testing

- [Kind](https://kind.sigs.k8s.io/) + [Kubeconform](https://github.com/yannh/kubeconform) + [Chainsaw](https://kyverno.github.io/chainsaw/) e2e

### Core Services

| Service | Solution | Notes |
|---------|----------|-------|
| DNS (Private) | [AdGuard Home](https://adguard.com/en/adguard-home/overview.html) | 2 instances (VM + RPi), OpenTofu provisioned |
| DNS (Public) | [DNSControl](https://dnscontrol.org/) → [Gcore](https://gcore.com/dns) + [deSEC](https://desec.io/) | CI-managed, redundant |
| Dynamic DNS | [ddns-updater](https://github.com/qdm12/ddns-updater) | Keeps public DNS records in sync |
| VPN | [Tailscale](https://tailscale.com/) | Mesh VPN connecting all sites |
| Network Controller | [UniFi Controller](https://ui.com/) | Manages WiFi APs |
| Auth | [Pocket ID](https://github.com/pocket-id/pocket-id) | OIDC provider |
| Secrets (K8s) | [Infisical](https://infisical.com/) | |
| Databases | [CNPG](https://cloudnative-pg.io/), [MariaDB Operator](https://github.com/mariadb-operator/mariadb-operator) | Crossplane compositions |
| Cache | [Valkey](https://valkey.io/) | Redis replacement, 3-node Sentinel HA |
| Container Registry | [Harbor](https://goharbor.io/) | 133 projects, 366 repos, ~200GB S3, 274 replication policies, Crossplane managed |
| Nix Cache | [NCPS](https://github.com/kalbasit/ncps) | Local caching proxy for Nix/Devbox, speeds up CI pipelines |
| Policy Engine | [Kyverno](https://kyverno.io/) | s3bkp auto-injection, resource defaults |
| Dependency Management | [Renovate](https://docs.renovatebot.com/) + [Renovate Operator](https://github.com/renovatebot/renovate-operator) | Per-repo RenovateJobs, central config, custom regex/groups |
| Observability | [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack), [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter) | Prometheus, Grafana, Alertmanager |
| Exporters | AdGuard, domain, Hetzner Cloud, NUT, Proxmox VE, MQTT, Tailscale | Fleet of custom metric exporters |
| Logging | [Loki](https://grafana.com/oss/loki/), [OpenTelemetry Operator](https://opentelemetry.io/docs/kubernetes/operator/) | Logs, instrumentation |
| Cert Monitoring | [certmon](https://github.com/mhkarimi1383/certmon) | TLS certificate expiry monitoring |
| Backups (K8s) | [Velero](https://velero.io/) + [Velero UI](https://github.com/otwld/velero-ui), [Kasten K10](https://www.kasten.io/), s3bkp (custom) | PVC snapshots, cross-cluster migration |
| Backups (Postgres) | [Barman](https://pgbarman.org/) (CNPG), [PGBackWeb](https://github.com/eduardolat/pgbackweb) | |
| Backups (VMs) | [Kopia](https://kopia.io/), [rsnapshot](https://rsnapshot.org/) | |
| Personal Cloud | [Nextcloud](https://nextcloud.com/) | File sync, photos, calendar, contacts |
| Home Automation | [Home Assistant](https://www.home-assistant.io/) | |
| Automation | [n8n](https://n8n.io/) | Workflow automation |
| Notifications | [ntfy](https://ntfy.sh/) | Push notifications for alerts and automations |
| Secrets (Personal) | [Vaultwarden](https://github.com/dani-garcia/vaultwarden) | Bitwarden server |
| Wiki | [Wiki.js](https://js.wiki/) | |
| URL Shortener | [Kutt](https://kutt.it/) | Self-hosted link shortening |
| RSS | [FreshRSS](https://freshrss.org/) + [RSS-Bridge](https://rss-bridge.org/) | Fiery Feeds on iOS |
| Bookmarks | [Linkwarden](https://linkwarden.app/) | |
| Resume | [Reactive Resume](https://rxresu.me/) | |
| Sharing | [PrivateBin](https://privatebin.info/), [Yopass](https://yopass.se/), [croc](https://github.com/schollz/croc), [transfer.sh](https://github.com/dutchcoders/transfer.sh), [uptermd](https://github.com/nicholasgasior/upterm) | Pastes, secrets, files, web terminal |

### Custom Solutions

| Tool | Description |
|------|-------------|
| s3bkp | K8s-native PVC backup/restore with cross-cluster migration, auto-injected via Kyverno |
| kcl-ci | CI/CD workflow generator using KCL, self-regenerating workflows (38+ repos) |
| git-manager | TUI for managing Git repos (clone, pull, push, PRs, mirrors, CI status) |
| imdbtop250rss | IMDb Top 250 to RSS feed |
| theme-api | Theme sync across devices |

### AI Tooling

Most work done with [Claude Code](https://claude.ai/claude-code) + custom prompts

| MCP | Purpose |
|-----|---------|
| [dot-ai]({{< ref "ai-stuff/dot-ai" >}}) | K8s deployments, remediation, cluster queries |
| [Grafana](https://github.com/grafana/mcp-grafana) | Dashboards, alerts, incidents, Loki/Prometheus queries |
| [Prometheus](https://github.com/pab1it0/prometheus-mcp) | Direct Prometheus queries |
| [Context7](https://github.com/upstash/context7) | Library documentation lookup |

---

## DR Site

### Hardware

**Synology DS1621+**

- 32 GB RAM
- 3x Seagate IronWolf Pro 14TB
- 2x Synology 400GB M.2 (cache)

### Purpose

- Backup target for [Kopia](https://kopia.io/)
- Offsite backups and replicas

---

## Cloud / VPS

### Hardware

**[Hetzner CX32](https://www.hetzner.com/cloud/)**

- 4 vCPU (AMD), 8 GB RAM, 80GB
- 75GB extra storage

### Services

- **Mail:** Postfix, Dovecot, Roundcube, PostfixAdmin *(yes, I self-host email in 2026 — I like pain)*
- **Web:** WordPress sites, MariaDB
- **Reverse Proxy:** [Traefik](https://traefik.io/)
- **Utilities:** echoip, [bore](https://github.com/ekzhang/bore) (tunnel), tinyproxy
- **Monitoring:** Hetzner-exporter, node-exporter

