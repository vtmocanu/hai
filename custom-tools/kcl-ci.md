# kcl-ci: DRY Forgejo Workflows in KCL (and Why I'm Sunsetting It)

<img src="/images/kcl-ci.png" class="hero-image" alt="A friendly cartoon robot in a homelab control room holding a glowing KCL blueprint scroll that fans out into STEPS, JOBS, and WORKFLOWS layers, condensing into YAML documents, with a Forgejo pipeline graph showing expansion on the right" style="max-width: 700px; width: 100%; height: auto;" />

{{< callout type="info" >}}
**TL;DR.** I had ~30 repos with near-identical Forgejo workflow YAML. Reusable workflows existed, but the nested-run UX was ugly and they only let me share whole workflows, not steps or jobs. I wrote a small KCL library, `kcl-ci`, that exposes typed `steps`, `jobs`, and `workflows`, and a tiny composite action that re-renders the YAML on every push and auto-commits the result. It worked. Then three things lined up: I moved my Crossplane compositions [off KCL onto custom Go functions](/crossplane/kcl-to-go-functions/), KCL's contributor momentum cooled, and Forgejo v15 shipped real reusable-workflow expansion. So `kcl-ci` is being retired in favour of plain reusable workflows. This post is the story of the tool, the wider DRY-CI landscape (GitLab, Argo, GitHub, Forgejo), and what I'd do differently.

**If you're on GitHub:** you don't get Forgejo v15's expansion. Watch the [Rawkode CUE video](https://www.youtube.com/watch?v=MFtQAhIqzco) and build the CUE version of my `kcl-ci` implementation, which is a small library of typed `steps`, `jobs`, and `workflows` plus a tiny action that re-renders them into committed YAML on every push. Renovate watches the central library and opens an update PR in every consumer repo whenever you touch it, so a single change fans out to your whole fleet automatically. That's still the cleanest path on GitHub today.
{{< /callout >}}

## How the idea started

I'd been carrying this idea around for a while: write CI pipelines as typed, validated configuration, not as YAML you copy between repos until they slowly drift. The idea got a lot more concrete when I watched David Flanagan's [Replace Your GitHub Actions YAML with CUE](https://www.youtube.com/watch?v=MFtQAhIqzco) on Rawkode Academy. The video is about [CUE](https://cuelang.org/), not [KCL](https://kcl-lang.io/), but the shape of the argument is the same: typed config, schemas, composition, generate the YAML at build time.

At that point I was already neck-deep in [KCL for Crossplane compositions](/crossplane/why-crossplane/). The toolchain was installed, the repos were structured around it, and Claude was doing most of the actual KCL writing for me anyway. Picking up a second typed-config DSL for CI when Claude could just keep writing KCL felt silly. So I built `kcl-ci` in KCL instead of CUE.

## The problem this was solving

I run roughly 30 repos in my Forgejo instance: container apps, Helm charts, OpenTofu, Packer templates, DNSControl, Ansible projects, [K8s prepuller configs](/custom-tools/prepuller/). Each one wants a CI pipeline that, in broad strokes, does the same thing: checkout, load secrets from Infisical, security scans (KICS, Checkov, Hadolint, Trivy, Grype), build, push, scan again, sometimes semantic-release.

The first version was honest copy-paste. Every new repo cloned a workflow from "the repo most similar to this one" and edited the parts that differed. After a while I had:

- Hadolint pinned to seven different versions across the fleet.
- Three slightly different ways to log into Harbor.
- A bug I'd fixed in one repo's KICS step that was still broken in eleven others.
- And a folder full of `.forgejo/workflows/*.yml` files that nobody, including me, was confident they understood end to end.

Forgejo at the time had `uses: org/repo/.forgejo/workflows/foo.yml@ref` (the same reusable-workflows model as GitHub), but the UX was rough (more on that below) and even when it worked, it didn't help me share *steps* and *jobs*, only entire workflows. I needed something more granular.

## What `kcl-ci` does

`kcl-ci` is two things in one repo:

1. **A KCL library** of typed templates organised in three layers (`steps/`, `jobs/`, `workflows/`), plus a `versions.k` file that pins every action version with Renovate-readable comments.
2. **A composite Forgejo Action** (`action.yml`) that consumer repos invoke from a tiny self-managing workflow. On every push it runs `kcl run`, regenerates the YAML files under `.forgejo/workflows/`, and auto-commits any diff with `[skip ci]`.

The architecture is deliberately boring:

```treeview
kcl-ci/
├── schemas/
│   ├── config.k       # Global config (git host, runners, Infisical project)
│   └── helpers.k      # Workarounds for KCL's ${{ }} interpolation problem
├── steps/             # Atomic step factories (checkout, infisical, build_push, ...)
├── jobs/              # Jobs that compose steps (security_scans, container_build, ...)
├── workflows/         # Whole workflows (container_build, tofu, prepuller, ...)
├── versions.k         # Centralised action versions, Renovate-tracked
└── action.yml         # Composite action consumed by every repo
```

### Layer 1: steps

A step is the smallest unit, a single `uses:` or `run:` block. Here's the real `checkout` step factory, lightly trimmed:

```kcl
checkout = lambda config: cfg.Config = cfg.DEFAULT_CONFIG,
                  secrets: cfg.Secrets = cfg.DEFAULT_SECRETS -> {str:} {
    {
        name = "Checkout"
        uses = _action_url(config.xternal_repo, "checkout",
                           versions.CHECKOUT_VERSION, config, secrets)
    }
}
```

The factory pattern lets me override the runner, the git host, or the token without forking the step. Every consumer gets the same checkout, pinned to the same version, fed from the same `versions.k`.

### Layer 2: jobs

A job composes steps and adds the surrounding scaffolding (runner, permissions, `needs`, `if` conditions, outputs). The `security_scans` job, for example, fans out into KICS, Checkov, Trivy filesystem, Grype filesystem, and Hadolint, each with its own caching strategy and severity gates.

### Layer 3: workflows

A workflow stitches jobs into a complete pipeline. Here's a stripped-down consumer `main.k` for a container repo:

```kcl
import kcl_ci.schemas.helpers as h
import kcl_ci.workflows.kcl_ci_build as kcl_build_mod
import kcl_ci.workflows.container_build as build_mod
import kcl_ci.workflows.container_release as release_mod

# Self-managing workflow that regenerates the others on every push
_kcl_workflow = kcl_build_mod.kcl_ci_build_workflow()
h.dump_workflow(_kcl_workflow, "../workflows/kcl-ci-build.yml")

# Build pipeline
_build_config = build_mod.ContainerBuildConfig {
    harbor_project = "wxs"
    enable_helm_chart = True
}
_build = build_mod.container_build_workflow(_build_config)
h.dump_workflow(_build, "../workflows/build.yml")

# Release pipeline
_release = release_mod.container_release_workflow(
    release_mod.ContainerReleaseConfig {
        required_checks = build_mod.required_checks(_build_config)
        enable_helm_chart = True
    }
)
h.dump_workflow(_release, "../workflows/release.yml")
```

About 20 lines of KCL produce three multi-hundred-line YAML files. The consumer never touches the generated `.yml` directly. If they try, the next push regenerates them.

### The self-managing trick

The neat-but-slightly-cursed part is that one of the workflows `kcl-ci` generates is the workflow that runs `kcl-ci`. Every consumer repo has a `.forgejo/workflows/kcl-ci-build.yml` that:

1. Checks out the repo.
2. Runs the `kcl-ci` composite action.
3. Auto-commits regenerated workflows with `[skip ci]` so it doesn't loop.

When I bump `kcl-ci` itself, Renovate opens a PR in every consumer; merging it triggers a regeneration of all the workflows in that repo. No manual fan-out. That part actually held up beautifully.

## Where it bit me

It wasn't all roses.

**KCL's `${{ }}` problem.** GitHub-style expressions clash with KCL's string interpolation. Anywhere I wanted to emit `${{ github.ref }}` I had to call a helper:

```kcl
gha_github = lambda prop: str -> str {
    "$" + "{{" + " github." + prop + " }}"
}
```

This is fine for a library author, ugly for casual contributors, and a constant low-level papercut.

**`.format()` brace-escaping pitfalls.** Anyone who's hit KCL's gotchas knows the format strings don't behave like Python's. I lost an embarrassing amount of time to that.

**Generated YAML is not the source of truth, except when it is.** Reviewers can't diff a KCL change without also reviewing the YAML diff. I worked around it with a pre-commit-friendly `task kcl:build` and a CI check that fails on stale outputs, but the cognitive overhead never fully went away.

**Cross-instance is impossible.** A consumer on a different Forgejo (or on GitHub) can't pull a private KCL library without auth juggling. I worked around it with a `READ_REPO_TOKEN` injected into `kcl.mod` URLs, but it's the kind of thing that fails confusingly.

## The wider DRY-CI landscape

Building `kcl-ci` forced me to look hard at what everyone else does. Three reference points kept coming up.

### GitLab CI/CD Catalog: the one that got it right

GitLab shipped the [CI/CD Catalog](https://docs.gitlab.com/ci/components/) GA in [17.0, May 2024](https://about.gitlab.com/blog/2024/05/08/ci-cd-catalog-goes-ga-no-more-building-pipelines-from-scratch/). You publish typed, versioned "components" to a catalog, consumers pull them with `include:` and pass typed inputs, and crucially, **the included jobs render inline in the consuming pipeline**. There's no nested-run UX. The catalog is searchable. Inputs are validated. Versions are semver.

Having moved from GitLab CI to Forgejo Actions and GitHub Actions over the last few years, I'll just say it plainly: nothing in the GitHub-Actions ecosystem touches the GitLab CI/CD Catalog for DRY pipelines. GitLab is the clear winner for CI. It's not close.

### Argo Workflows: DRY by design, but it's a different beast

Argo Workflows has [`WorkflowTemplate`](https://argo-workflows.readthedocs.io/en/latest/workflow-templates/) and [`ClusterWorkflowTemplate`](https://argo-workflows.readthedocs.io/en/latest/cluster-workflow-templates/), referenced via `templateRef`. It's an excellent DRY model: templates are first-class Kubernetes resources, parameterised, composable, and the UI shows the full DAG inline. I've played with it and liked it. But Argo Workflows is a workflow engine that happens to be useful for CI, not a Git-native CI system. For "this branch was pushed, run these checks," it's overkill.

### GitHub Actions and Forgejo (pre-v15): reusable workflows, ugly UX

GitHub Actions has supported [reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows) for years. You can do `uses: owner/repo/.github/workflows/foo.yml@ref` and it works. **But the UI renders the called workflow as a separate, nested run** that you have to click into. The caller shows a single black-box step, the callee renders elsewhere. Composite actions collapse nicely; reusable workflows don't. It works, but it looks ugly, and it never feels integrated. Forgejo, until very recently, behaved exactly the same way: reusable workflows existed, but they were nested-run citizens.

This nested-run UX is the main reason I never seriously used GitHub-Actions-style reusable workflows for my fleet. The whole point of DRY is *visibility plus consolidation*; if I have to chase a run across three tabs to debug it, I've traded one problem for another. KCL templates rendered into local YAML kept everything visible.

### Forgejo v15: expansion lands

Then [Forgejo v15.0](https://forgejo.org/2026-04-release-v15-0/) shipped on 16 April 2026 with the thing I'd been waiting for: when a caller workflow omits the top-level `runs-on`, **the called workflow's inner jobs are expanded into the parent run's job graph** as siblings. Same UI, same job list, same logs view. No more clicking through nested runs. The work was tracked from issue [#9768](https://codeberg.org/forgejo/forgejo/issues/9768) and went through a few iterations (PR [#10448](https://codeberg.org/forgejo/forgejo/pulls/10448) was closed in favour of the simpler PR [#10525](https://codeberg.org/forgejo/forgejo/pulls/10525), which is what actually shipped), with runner support in v12.2.0.

There are caveats: it's [public repos only, same Forgejo instance only](https://forgejo.org/2026-04-release-v15-0/), no cross-instance or private callers. For my homelab, where everything lives on one Forgejo, that's exactly fine.

The headline: **Forgejo just shipped, natively, the thing GitHub still doesn't do.** And it does it without me having to maintain a code-generation library on the side.

## Why `kcl-ci` is being retired

Three things lined up at the same time:

1. **I migrated my Crossplane compositions [from KCL to custom Go functions](/crossplane/kcl-to-go-functions/) and got an 800x CPU reduction.** Once KCL stopped being part of my Crossplane stack, the "I already use it, so why not" argument for picking KCL over CUE evaporated. The remaining KCL footprint was just `kcl-ci`.
2. **KCL's momentum has cooled.** It's still a [CNCF Sandbox project](https://www.cncf.io/projects/kcl/) and the last release ([v0.11.2](https://github.com/kcl-lang/kcl/releases) on 18 April 2025) wasn't that long ago, so "dead" overstates it. But the release cadence has clearly slowed, and overall project activity (contributors, stars, forks) appears to have cooled noticeably year over year. It's not the bet I'd make today.
3. **Forgejo v15 makes the problem `kcl-ci` solves smaller.** With reusable workflow expansion, I can move shared logic into a few `.forgejo/workflows/*.yml` files in a central repo, reference them with `uses:`, and the UI keeps everything inline. The DRY win is now native.

So the plan is: migrate the workflows that benefit most from native reusable workflows first (security scans, container build, semantic release), keep `kcl-ci` running for the long tail until the migration is done, then delete it. **Update:** that migration is done. The successor is [`fj-ci`](/custom-tools/fj-ci/), a library of hand-written reusable workflows with no codegen at all.

## Lessons I'm taking with me

A few things I'd keep, even on the next iteration.

- **Centralised version pinning with Renovate-readable comments was the single biggest win.** Even if I drop KCL, the `versions.k` pattern (or its equivalent) is non-negotiable.
- **Self-managing workflows are worth the small spook-factor.** A pipeline that regenerates itself on push is delightful once it works, and it removes the "did you remember to rerun the generator" failure mode entirely.
- **Generated YAML is fine as long as the diff is reviewable.** What hurt wasn't the generation, it was the language-specific gotchas (`${{ }}` interpolation, `.format()` braces). A simpler DSL or native reusable workflows avoid that entirely.
- **Don't out-clever the platform.** I built `kcl-ci` because Forgejo couldn't do what I needed at the time. Forgejo can do it now.

And the bigger lesson, the one I keep relearning: every time I move *away* from GitLab CI, I miss it more. GitLab got DRY pipelines right years ago. Everyone else is still catching up.

