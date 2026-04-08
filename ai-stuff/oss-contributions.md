# Co-Authored with AI

I am not a programmer. I have spent years operating Linux and Kubernetes infrastructure, but I cannot write production code on my own to a quality I'd feel comfortable shipping to a maintainer. What changed in the last year is that I can ship code anyway: Claude Code writes what I cannot, I keep ownership of the judgment calls, and we end up with patches and bug reports real maintainers actually act on.

This page is a running log of upstream OSS contributions — bug fixes, new features, and bug reports too detailed to ignore — where, **without Claude Code, I would not have filed it at all**. Either I would not have found the bug, traced it to the right code path, designed the feature, written the code, written the tests, read the upstream's lint config, or some combination of those.

## Workflow in one paragraph

For every contribution below — bug fix or new feature — I follow the same loop, captured in a private `upstream-fork` [skill]({{< ref "commands-vs-mcp-vs-skills" >}}): open an issue on my own tracker, draft a [PRD]({{< ref "dot-ai#prds-planning-before-implementing" >}}), fork the upstream repo, develop the change on a branch, run multi-agent code review with separate Claude subagents (QA, coding standards, code/logic), build and deploy the fork image to my cluster (when applicable), validate end-to-end against my real workload, then open the upstream PR.

I usually let maintainers know in private, or include something like this in the PR body:

> Full disclosure: I don't have any programming skills so I asked Claude Code to implement this and did my best to test it and not create AI slop.

## Contributions

### Features

#### k8s-cleaner #556 — Embedded web dashboard for scan results and on-demand triggers

<img src="/images/oss-k8s-cleaner-556.png" alt="k8s-cleaner web dashboard showing scan results, cleaner cards with Lua scripts, and trigger buttons" style="max-width: 800px; width: 100%; height: auto;" />

- **Project**: [gianlucam76/k8s-cleaner](https://github.com/gianlucam76/k8s-cleaner) — Kubernetes operator that uses Lua scripts to identify orphaned/unused resources
- **PR**: [gianlucam76/k8s-cleaner#556](https://github.com/gianlucam76/k8s-cleaner/pull/556)
- **Status**: merged 2026-04-05
- **Language**: Go + Preact

Added an embedded web dashboard behind a `--enable-web` flag, served by a Go HTTP server with a Preact + Tailwind SPA bundled into the binary via `embed.FS` (89KB total). The dashboard exposes a REST API at `/api/v1/` and lets you browse scan results, view the Lua scripts each cleaner runs, and trigger on-demand scans without `kubectl` access. Includes a read-only mode middleware, Helm chart values for all the new flags, and 24 new tests (16 Go handler tests + 8 Vitest UI component tests). Zero changes to the existing controller, API, or `pkg/` — all new code in `internal/web/` and `web/`.

#### renovate-operator #239 — PR activity per run in the operator UI

<img src="/images/oss-renovate-operator-239.png" alt="Renovate Operator dashboard showing PR activity column with expandable per-PR details" style="max-width: 800px; width: 100%; height: auto;" />

- **Project**: [mogenius/renovate-operator](https://github.com/mogenius/renovate-operator) — Kubernetes operator for Renovate
- **PR**: [mogenius/renovate-operator#239](https://github.com/mogenius/renovate-operator/pull/239)
- **Status**: merged 2026-04-02
- **Language**: Go + UI

Extended the Renovate log parser to detect 7 message types and extract per-PR activity (created, updated, unchanged, automerged) from the JSON logs of each Renovate run, then surfaced it in the operator dashboard with expandable accordion rows showing per-PR details and clickable links back to the forge. Added new CRD types (`PRAction`, `PRDetail`, `PRActivity`), a per-project status update flow, multi-forge URL handling (GitHub, Forgejo, GitLab), 25 new parser test cases, and a bonus deep-copy bugfix for `RenovateJobList`. Closes upstream issue #115.

### Bugs

#### pocket-id #1413 — Custom logos and favicons disappeared after every pod restart

- **Project**: [pocket-id/pocket-id](https://github.com/pocket-id/pocket-id) — self-hosted OIDC provider
- **PR**: [pocket-id/pocket-id#1413](https://github.com/pocket-id/pocket-id/pull/1413)
- **Issue**: [pocket-id/pocket-id#1407](https://github.com/pocket-id/pocket-id/issues/1407)
- **Status**: open, waiting for upstream review
- **Language**: Go

Pocket-ID's S3 backend returned full prefixed keys from List instead of relative paths, double-prefixing every subsequent Open and silently 404'ing — which made every custom logo and favicon vanish from my homelab after every pod restart. Fixed by stripping the prefix in List with a small `pathFromKey` helper plus round-trip unit tests.

### Issues

#### flux-operator #677 — Run Job button missing on the workloads page

- **Project**: [controlplaneio-fluxcd/flux-operator](https://github.com/controlplaneio-fluxcd/flux-operator) — controller for managing Flux CD
- **Issue**: [controlplaneio-fluxcd/flux-operator#677](https://github.com/controlplaneio-fluxcd/flux-operator/issues/677)
- **Status**: issue filed
- **Language**: Go

The Flux Operator web UI was hiding the "Run Job" button on workloads despite the user having permission to trigger them. Tracing the full flow (frontend → API → RBAC check → tests) revealed that `resource.go` was checking workload actions like `restart` against the wrong API group, and that the test suite was masking the bug with mock data.

#### renovate-operator #114 — UI showed every repo as not onboarded

- **Project**: [mogenius/renovate-operator](https://github.com/mogenius/renovate-operator) — Kubernetes operator for Renovate
- **Issue**: [mogenius/renovate-operator#114](https://github.com/mogenius/renovate-operator/issues/114)
- **Status**: fixed by maintainers in [v2.4.1](https://github.com/mogenius/renovate-operator/releases/tag/v2.4.1)
- **Language**: Go

The Renovate Operator UI was showing "No Config (renovate not onboarded)" for every repo despite all of them being fully onboarded. The log parser was using a naive `strings.Contains("onboarding")` that matched debug messages like `checkOnboarding()` present in every run, falsely reporting all repos as un-onboarded.

#### renovate-operator #117 — Onboarding detection still broken after the v2.4.1 fix

- **Project**: [mogenius/renovate-operator](https://github.com/mogenius/renovate-operator) — Kubernetes operator for Renovate
- **Issue**: [mogenius/renovate-operator#117](https://github.com/mogenius/renovate-operator/issues/117)
- **Status**: issue filed with Go reproducer
- **Language**: Go

After the v2.4.1 fix landed, large repos still showed as un-onboarded. The cause: Renovate emits a `"packageFiles with updates"` line that can be 190KB+, while Go's `bufio.Scanner` silently stops at 64KB — so the parser never reached the "Repository finished" marker at the end of the logs. One-line fix: `scanner.Buffer(make([]byte, 0), 1024*1024)`.

