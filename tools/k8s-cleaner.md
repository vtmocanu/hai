# k8s-cleaner: Finding Orphaned Resources in Kubernetes

<img src="/images/k8s-cleaner-logo.png" alt="k8s-cleaner logo" style="max-width: 200px;">

## The Problem

Kubernetes clusters accumulate junk over time. ConfigMaps nobody references, Secrets left behind after deleting an app, PVCs that aren't mounted anywhere, Helm releases someone installed manually and forgot about. In a GitOps-managed cluster, this is even more annoying because you expect everything to be tracked and accounted for.

You could write scripts to find these things, but then you need to maintain those scripts, schedule them, handle edge cases, and somehow get notified about the results.

## Enter k8s-cleaner

[k8s-cleaner](https://github.com/gianlucam76/k8s-cleaner) is an open-source Kubernetes controller by [Gianluca Mardente](https://www.linkedin.com/in/gianlucamardente/), part of the [Projectsveltos](https://github.com/projectsveltos) ecosystem. It lets you define scan policies using Lua scripts that run on a cron schedule, checking for resources that match your criteria.

What makes it stand out:

- **Lua-based policies** - You write the detection logic yourself, so you can check anything: labels, annotations, ownerReferences, cross-resource relationships
- **Scan-only mode** - You can run it in `action: Scan` mode where it only reports findings without deleting anything
- **Report CRs** - Results are stored as `Report` custom resources, queryable with kubectl
- **Slack notifications** - Optional webhook integration for alerts
- **Prometheus metrics** - Exposes `k8s_cleaner_scan_resources_total` for alerting on findings

The maintainer, Gianluca, is very responsive. Feature requests get picked up quickly, bugs get fixed promptly. It's the kind of project where you feel confident opening an issue because you know someone is actively paying attention.

## How I Learned to Respect `action: Delete`

A quick story about why I run everything in scan-only mode.

When I first set up k8s-cleaner, I was writing a policy to find unused Secrets. I copy-pasted from one of the examples in the documentation, tweaked the Lua logic, and deployed it. What I didn't notice: the example had `action: Delete`. Not `action: Scan`. Delete.

The next time the schedule fired, k8s-cleaner did exactly what I told it to do. It found all the Secrets matching my (not yet fully tuned) policy and deleted them. Including database credentials, TLS certificates, and Infisical-managed secrets across multiple namespaces. The important ones.

What followed was a fun evening of restoring secrets from Infisical, re-triggering cert-manager issuance, and restarting half the cluster. Everything was recoverable, but it took some work. Lesson learned: always double-check the `action` field.

I [opened an issue](https://github.com/gianlucam76/k8s-cleaner/issues/397) suggesting that new Cleaner CRs should default to `Scan` instead of `Delete`, so that copy-paste mistakes like mine would be safer by default. Because let's be honest, if you're writing a new policy, you probably want to see what it matches before it starts deleting things.

## How I Use It

So yes, I run almost all cleaners in **scan-only mode**. Nothing gets auto-deleted. The one exception is a `krelay-delete` cleaner that removes leftover ConfigMaps and Services from [krelay](https://github.com/knight42/krelay) sessions, which are safe to clean up automatically. Reports come in, I review them, and I either add exclusion rules or clean up manually. Here's the full list of scans, staggered at 5-minute intervals between 6 AM and 10 PM:

| Scan | What It Detects |
|------|-----------------|
| `unused-configmaps` | ConfigMaps not referenced by any Pod, Deployment, StatefulSet, or CronJob |
| `unused-secrets` | Secrets not referenced by any workload or Ingress TLS |
| `secrets-non-infisical` | Secrets not managed by Infisical (my secrets manager) |
| `pvc-scan` | PVCs not mounted by any Pod |
| `deployment-with-zero-replicas` | Deployments scaled to zero |
| `deployments-not-gitops` | Deployments not managed by Flux |
| `cnpg-orphan-resources` | CNPG ScheduledBackups/Backups referencing non-existing Clusters |
| `cnpg-orphan-prometheusrules` | PrometheusRules referencing non-existing CNPG Clusters (causes false alerts) |
| `helm-not-gitops` | Helm releases deployed manually, not via Flux |

## Configuration Examples

Each scan is a `Cleaner` CR with a schedule, resource selectors, and a Lua `evaluate()` function. Here are a few representative ones.

### Detecting Non-GitOps Deployments

This one checks if a Deployment is managed by Flux (controller labels). If not, it's flagged:

```yaml
apiVersion: apps.projectsveltos.io/v1alpha1
kind: Cleaner
metadata:
  name: deployments-not-gitops
spec:
  schedule: "25 6-22 * * *"
  action: Scan
  resourcePolicySet:
    resourceSelectors:
      - kind: Deployment
        group: "apps"
        version: v1
        namespaceSelector: "kubernetes.io/metadata.name notin (kube-system,kube-public,kube-node-lease)"
        evaluate: |
          function evaluate()
            hs = {}
            hs.matching = false

            local labels = obj.metadata.labels or {}

            local has_flux = labels["helm.toolkit.fluxcd.io/name"] ~= nil or
                             labels["kustomize.toolkit.fluxcd.io/name"] ~= nil

            if not has_flux then
              hs.matching = true
              hs.message = string.format(
                "Deployment '%s' in namespace '%s' is not managed by Flux",
                obj.metadata.name, obj.metadata.namespace)
            end

            return hs
          end
  notifications:
    - name: report
      type: CleanerReport
```

The `evaluate` function runs per-resource. You return `hs.matching = true` to flag it.

### Detecting Manually Installed Helm Releases

Flux's helm-controller sets `manager: "helm-controller"` in `managedFields`. A manual `helm install` sets `manager: "Helm"`. This scan checks for the difference:

```yaml
apiVersion: apps.projectsveltos.io/v1alpha1
kind: Cleaner
metadata:
  name: helm-not-gitops
spec:
  schedule: "40 6-22 * * *"
  action: Scan
  resourcePolicySet:
    resourceSelectors:
      - kind: Secret
        group: ""
        version: v1
        labelFilters:
          - key: owner
            operation: Equal
            value: helm
          - key: status
            operation: Equal
            value: deployed
        evaluate: |
          function evaluate()
            hs = {}
            hs.matching = false

            local managedFields = obj.metadata.managedFields or {}
            local isFluxManaged = false

            for _, field in ipairs(managedFields) do
              if field.manager == "helm-controller" then
                isFluxManaged = true
                break
              end
            end

            if not isFluxManaged then
              hs.matching = true
              hs.message = string.format(
                "Helm release '%s' in namespace '%s' is not managed by Flux",
                obj.metadata.labels["name"] or "unknown", obj.metadata.namespace)
            end

            return hs
          end
  notifications:
    - name: report
      type: CleanerReport
```

### Cross-Resource Checks: Orphaned CNPG PrometheusRules

Some scans need to correlate multiple resource types. This one fetches both PrometheusRules (with CNPG labels) and CNPG Clusters, then checks if the clusters referenced in alert rules actually exist. Orphaned PrometheusRules cause false `CNPGClusterOffline` critical alerts:

```yaml
apiVersion: apps.projectsveltos.io/v1alpha1
kind: Cleaner
metadata:
  name: cnpg-orphan-prometheusrules
spec:
  schedule: "35 6-22 * * *"
  action: Scan
  resourcePolicySet:
    resourceSelectors:
      - kind: PrometheusRule
        group: "monitoring.coreos.com"
        version: v1
        labelFilters:
          - key: app.kubernetes.io/part-of
            operation: Equal
            value: cloudnative-pg
      - kind: Cluster
        group: "postgresql.cnpg.io"
        version: v1
    aggregatedSelection: |
      function evaluate()
        local hs = {}
        local clusters = {}
        local orphaned = {}

        -- Index existing clusters
        for _, resource in ipairs(resources) do
          if resource.kind == "Cluster" then
            local key = resource.metadata.namespace .. ":" .. resource.metadata.name
            clusters[key] = true
          end
        end

        -- Check each PrometheusRule
        for _, resource in ipairs(resources) do
          if resource.kind == "PrometheusRule" then
            local referencedClusters = {}
            local hasRef = false

            for _, group in ipairs(resource.spec.groups or {}) do
              for _, rule in ipairs(group.rules or {}) do
                if rule.labels and rule.labels.cnpg_cluster then
                  hasRef = true
                  local key = resource.metadata.namespace .. ":" .. rule.labels.cnpg_cluster
                  referencedClusters[key] = true
                end
              end
            end

            if hasRef then
              local anyExists = false
              for key, _ in pairs(referencedClusters) do
                if clusters[key] then anyExists = true; break end
              end
              if not anyExists then
                table.insert(orphaned, {resource = resource})
              end
            end
          end
        end

        if #orphaned > 0 then hs.resources = orphaned end
        return hs
      end
  notifications:
    - name: report
      type: CleanerReport
```

Notice the difference: single-resource scans use `evaluate` inside a `resourceSelector` (checking `obj`), while cross-resource scans use `aggregatedSelection` at the `resourcePolicySet` level (iterating over `resources`).

## Exclusions

Every scan checks for a global ignore annotation first:

```yaml
metadata:
  annotations:
    k8s-cleaner.wxs.io/ignore: "true"
```

Beyond that, each scan has Lua-based exclusion logic for known patterns: system namespaces, operator-managed resources, specific labels, ownerReferences. The unused-configmaps and unused-secrets scans have the longest exclusion lists because there are many legitimate reasons a ConfigMap or Secret exists without being directly referenced by a Pod.

## Reviewing Reports

Results are stored as cluster-scoped `Report` CRs:

```bash
# List all reports
kubectl get reports

# Check a specific report
kubectl get report unused-configmaps -o json | \
  jq '.spec.resourceInfo[] | .resource | fromjson | {kind, namespace, name}'
```

Each flagged resource is stored in `.spec.resourceInfo[].resource` as a JSON string containing `apiVersion`, `kind`, `name`, and `namespace`.

## Prometheus Alerts

I also have PrometheusRules that fire when scans detect issues that persist beyond a threshold:

| Alert | Severity | Duration |
|-------|----------|----------|
| Unused ConfigMaps/Secrets | info | 2h |
| Orphaned PVCs | warning | 2h |
| Non-Flux Deployments | warning | 1h |
| Orphaned CNPG Resources | warning | 1h |
| Non-GitOps Helm Releases | warning | 1h |

The scans find things. The alerts make sure I actually deal with them.

## Final Thoughts

k8s-cleaner fills a gap that most cluster operators deal with using ad-hoc scripts or just ignore entirely. The Lua-based approach is flexible enough to encode any detection logic, and scan-only mode means you can deploy it without worrying about accidental deletions.

If you run a GitOps-managed cluster and want visibility into what's drifting or accumulating, give it a look.

