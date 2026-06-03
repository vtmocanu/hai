# fj-ci: DRY Forgejo Workflows

<img src="/images/fj-ci.png" alt="A friendly cartoon robot slotting a glowing YAML card into a wall-mounted fj-ci workflow library panel that fans out light traces to a grid of repository tiles, with a pipeline graph showing called workflows rendering inline as sibling jobs, and the retired kcl-ci blueprint scroll displayed in a glass museum case" class="hero-image" style="max-width: 700px; width: 100%; height: auto;" />

{{< callout type="info" >}}
**TL;DR.** When I [retired kcl-ci](/custom-tools/kcl-ci/), the plan was to replace the KCL code-generation layer with plain Forgejo reusable workflows, now that v15 expands them inline. That replacement is `fj-ci`: one repo with ~25 hand-written `workflow_call` workflows, consumed by 85+ repos through thin wrappers pinned to semver tags. Renovate fans out version bumps, semantic-release cuts tags automatically on every push to main, and a small lint quartet turns every expansion bug I hit into a rule. No codegen, no DSL, no committed generated YAML. This is the kcl-ci idea evolved, with the platform finally doing the heavy lifting.
{{< /callout >}}

## Where the story left off

The [kcl-ci post](/custom-tools/kcl-ci/) ended with a plan. Forgejo v15.0 had shipped [reusable workflow expansion](https://forgejo.org/2026-04-release-v15-0/), so the typed-config codegen layer I had built to keep ~30 repos DRY was suddenly solving a smaller problem than it used to. I would migrate workload by workload, then delete kcl-ci.

That's done. kcl-ci is archived, and its native successor is `fj-ci`: same DRY goal, no code generation, the platform itself doing the composition.

## What fj-ci is

`fj-ci` is a single repo on my Forgejo instance holding hand-written reusable workflows, one YAML file per workload: container build and release, Helm, OpenTofu, Packer, DNSControl, Ansible, Hugo deploy, semantic-release, ISO builds, and so on. Two dozen consumer-facing workflows in total.

Every one of them follows the same contract:

- Triggered only by `on.workflow_call:` with **typed `inputs:`**.
- Required `secrets.*` reads documented in a header comment. Forgejo doesn't have typed `workflow_call` secrets yet, so the comment is the contract.
- No top-level `runs-on:`, so v15 expansion renders the inner jobs as siblings in the caller's job graph.

Here's the library side, condensed from the real container workflow (heavily simplified for clarity; the real one also runs lint and image vulnerability scans):

```yaml {filename=".forgejo/workflows/container.yml (fj-ci library)"}
name: Container

# Secrets contract (header comment, since workflow_call secrets aren't typed):
#   reads REGISTRY_USER / REGISTRY_PASS for the image push
on:
  workflow_call:
    inputs:
      registry-project:
        description: Registry project to push the image to
        type: string
        required: true
      vuln-warn-only:
        description: Downgrade vulnerability findings to warnings
        type: boolean
        default: false

# note: no top-level runs-on; that would collapse the inline expansion

jobs:
  build-and-push:
    runs-on: docker
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Build and push image
        run: ./build.sh "${{ inputs.registry-project }}"

  release:
    needs: build-and-push
    runs-on: docker
    # internal gate on PRESERVED github fields only;
    # never event_name, which is rewritten under expansion (see bugs below)
    if: github.ref == 'refs/heads/main' && github.event.pull_request == null
    steps:
      - name: Run semantic-release
        run: npx semantic-release
```

Note the `release` job's `if:`. One workflow serves both regimes, building everywhere but releasing only on main, decided entirely inside the library. The caller never tells it what event fired; the library never asks.

Consumers carry only a thin wrapper. The same container app's entire CI config is now roughly this (also simplified). **This is how I want every repo's workflow to look:**

```yaml {filename=".forgejo/workflows/build.yml (consumer repo)"}
name: build
on:
  push:
  pull_request:

jobs:
  ci:
    uses: my-org/fj-ci/.forgejo/workflows/container.yml@v0.12.0
    with:
      registry-project: my-app
    secrets: inherit
```

The wrapper owns the trigger semantics (`on:`, branch filters, `paths-ignore`), the pin owns the version, and everything else lives centrally. In the run UI, the called workflow's jobs show up inline as if they were written in the consumer repo. That inline rendering is the whole reason this works; it's also why the library repo has to be public and on the same Forgejo instance. That v15 constraint I can live with.

## The one rule that matters

Almost every design decision in fj-ci falls out of a single principle:

> A reusable workflow is a pure function of its `inputs:` and `secrets:`. Trigger-event semantics live in the caller's `on:` block, nowhere else.

No peeking at `github.event_name` inside the library, no consumer passing `${{ github.* }}` expressions across the call boundary, no caller-side `if:` deciding whether to expand. Inputs in, jobs out.

I didn't start with this rule. I extracted it, one bug at a time.

## The bugs, TL;DR version

The expansion boundary mangles more state than you'd expect. I hit each of these in production and root-caused it against the Forgejo source. The one-liners:

- `github.event_name` is rewritten to `workflow_call` inside expanded jobs, so event-name gates mis-evaluate on every real trigger. Other `github.*` fields (`ref`, `base_ref`, the event payload) survive intact and are safe to gate on.
- `${{ github.* }}` in a caller's `with:` evaluates to empty strings; `${{ secrets.* }}` there fails schema validation and the whole consumer workflow silently never runs.
- A caller-side `if:` on a `uses:` job is ignored. Expansion happens unconditionally at parse time.
- `concurrency:` declared inside a called workflow is not enforced across runs.
- Cross-job `if:` on `needs.*.outputs.*` doesn't evaluate.
- URL-form `uses: https://...` refs silently downgrade to the old nested-run UX; only short-form refs expand.

Each became a lint rule or a documented pattern the day I found it. That's the payoff of owning the library: a bug becomes a rule once, instead of a surprise thirty times.

## Guard rails

Four checks run on every push to fj-ci, and locally before every push:

1. **actionlint** for workflow correctness.
2. **zizmor** for security audit.
3. **gitleaks**, because the repo is public and a leaked secret would be exposed instantly.
4. A custom guard script that encodes the bug list above: it rejects URL-form self-references, job-level `if: github.event_name` gates, and `${{ secrets.* }}` smuggled into `with:` blocks. It scans the documented consumer examples too, which is exactly the vector that once leaked a broken pattern into two consumer repos.

The guard script also runs inside the release pipeline, after the release tooling rewrites files, so a corrupting rewrite aborts the release before the tag is cut.

## Releases without ceremony

kcl-ci taught me that [centralised version pinning with Renovate fan-out](/custom-tools/kcl-ci/#lessons-im-taking-with-me) was the single biggest win, so fj-ci doubles down on it:

1. Every push to main waits for CI to go green (the same runners and queue I watch with [fj-queue](/custom-tools/fj-queue/)), then runs [semantic-release](https://semantic-release.gitbook.io/).
2. Conventional commits decide the bump: `feat:` is minor, `fix:` is patch, breaking changes are major.
3. The prepare step writes the changelog, atomically rewrites every internal `@vX.Y.Z` self-reference to the new version, and re-runs the guard script on the result.
4. The `vX.Y.Z` tag is pushed, and Renovate opens (and auto-merges, for non-breaking bumps) the pin update in every consumer repo.

Manual tagging is retired. Cutting a release is now indistinguishable from merging a commit, and a version bump propagates to the whole fleet without me touching a single consumer.

One refinement worth noting: consumer repos are configured so that a pure fj-ci pin bump does *not* cut a release of the consumer itself. Early on, every fj-ci patch release triggered a pointless wave of downstream releases. Now the pin bump merges silently and the consumer only releases on its own changes.

## The migration, in numbers

- **1 pilot repo** migrated by hand to shake out the expansion bugs, then used as the template.
- **~30 kcl-ci consumers** migrated in batches grouped by workload type, each PR replacing the codegen setup with hand-written wrappers.
- **85+ repos** on fj-ci today (the fleet grew past the original kcl-ci population, since net-new repos now onboard with a single scaffolding task).
- **0 codegen**. Nothing renders YAML anymore; what's in the repo is what runs.

## What carried over from kcl-ci

Looking back at the [lessons list](/custom-tools/kcl-ci/#lessons-im-taking-with-me) from the retirement post, most of it survived the rewrite, just wearing different clothes:

- **Central pinning + Renovate** carried over directly and got stronger: semver tags on a library repo are even easier for Renovate to track than versions in a KCL file.
- **Reviewable diffs** are no longer something to engineer for. The YAML in each repo *is* the source, so every diff is the real change.
- **"Don't out-clever the platform"** turned out to be the headline. kcl-ci existed because Forgejo couldn't express DRY pipelines natively. The moment it could, the right amount of custom tooling dropped to roughly one lint script and a release config.

And yes, the GitLab line from the last post still stands. GitLab got DRY pipelines right years ago. But with v15 expansion plus a disciplined library repo, Forgejo is the closest I've ever been to not missing it.

