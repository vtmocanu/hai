# Claude Code Action for Forgejo

![Forgejo Claude Code Action Fork](/images/claude-code-action-forgejo.png)

Anthropic ships a [GitHub Action](https://github.com/anthropics/claude-code-action) that lets Claude Code respond to `@claude` mentions on issues and PRs (tag mode) or run tasks from a prompt input (agent mode). It's a solid piece of automation, but it's GitHub-only.

I wanted this on my [Forgejo](https://forgejo.org/) instance. Searched around, found nothing for Forgejo. A few projects exist for [Gitea](https://about.gitea.com/), but Forgejo has diverged enough that they didn't fit. So I built my own.

~~Fully vibe coded~~ Agentic coded. Works for my setup so far.

## TL;DR

**Repo:** [codeberg.org/vtmocanu/claude-code-action](https://codeberg.org/vtmocanu/claude-code-action)

**What it is:** A fork of [Anthropic's claude-code-action](https://github.com/anthropics/claude-code-action) that works on Forgejo (and Forgejo Actions).

**How to use it:** Point your Forgejo workflow at the Codeberg repo instead of the GitHub one. Add your Anthropic API key (or Claude Code OAuth token if you're on a subscription) as a secret, create a dedicated bot user, and you're set. The [Forgejo setup guide](https://codeberg.org/vtmocanu/claude-code-action/src/branch/main/docs/forgejo.md) walks through everything.

**How I use it:** Claude reviews my PRs, answers questions on issues, and runs automated tasks (upstream syncs, code generation) across my Forgejo repos. Tag mode (`@claude` mentions) and agent mode (`prompt` input) both work.

**How it differs from upstream:**
- Replaces GitHub's OIDC/App token exchange with direct token auth
- Swaps GraphQL queries for parallel REST calls (Forgejo has no GraphQL)
- Adds platform auto-detection (explicit input, `FORGEJO` env var, or URL heuristic)
- Adds bot identity inputs (`bot_id`, `bot_name`, `bot_email`) since Forgejo has no built-in bot accounts
- Prettified CI output (scannable log lines instead of raw JSON)
- Fixes `allowed_tools`/`disallowed_tools` inputs that were broken in upstream ([#181](https://github.com/anthropics/claude-code-action/issues/181), [#287](https://github.com/anthropics/claude-code-action/issues/287), [#724](https://github.com/anthropics/claude-code-action/issues/724), [#690](https://github.com/anthropics/claude-code-action/issues/690))

Everything else is upstream code running unchanged.

If you're among the 50 or so Forgejo users out there 😄 and you give this a try, ping me on [LinkedIn](https://www.linkedin.com/in/vtmocanu/) and let me know how it goes. **I'm happy to get other maintainers on board too.** Right now it's a one-person fork, and more eyes on the upstream sync and Forgejo edge cases would go a long way.

## The Adapter Pattern

The upstream action is deeply integrated with GitHub: OIDC token exchange, GraphQL queries, GitHub App authentication. None of that exists on Forgejo.

Instead of rewriting everything, I used a lean adapter pattern. Only 4 areas have genuinely Forgejo-specific code:

**Platform detection** uses a three-tier strategy:
1. Explicit `platform: "forgejo"` input (highest priority)
2. `FORGEJO` env var (set by Forgejo runner v7.0+)
3. `GITHUB_SERVER_URL` heuristic: non-`github.com` URL means Forgejo

**Token handling** is simpler on Forgejo. GitHub uses OIDC exchange to get an app token; Forgejo just uses the token directly.

**Data fetching** replaces GitHub's 3 GraphQL queries with 5-8 parallel REST calls. Forgejo's API is Octokit-compatible for REST, but has no GraphQL.

**Comment formatting** accounts for URL pattern differences. Forgejo uses `/actions/runs/{RUN_NUMBER}` instead of `/runs/{RUN_ID}`, `?expand=1` instead of `?quick_pull=1` for PR creation, and a Unicode spinner instead of a CDN-hosted GIF.

**Bot identity** is another Forgejo-specific concern. GitHub has the built-in `claude[bot]` account for commit attribution. Forgejo doesn't have that concept, so you create a dedicated `claude` user on your instance. The action inputs `bot_id`, `bot_name`, and `bot_email` control git commit attribution and self-trigger prevention.

Everything else in the GitHub modules works on both platforms as-is: permissions, comments, branches, git config, prompt formatting. The key files that have platform branches use an early-return pattern to minimize upstream merge conflicts:

```typescript
if (platform === "forgejo") {
  githubToken = setupForgejoToken();
} else {
  githubToken = await setupGitHubToken();
}
```

Forgejo-specific code sits at the top of functions, before the GitHub path, so upstream changes to the GitHub code rarely conflict.

## Custom Changes

I did make a few changes beyond core Forgejo support:

**Prettified CI output.** The upstream action dumps raw JSON to CI logs when `show_full_output` is enabled. I added a formatter that turns it into scannable log lines:

```text
🚀 Claude Code v2.1.61 (claude-3-5-sonnet-20241022) — 15 tools, 2 MCP servers
💬 "I'll analyze the PR changes and provide feedback."
🔧 Grep: "async function" in src/
   ↳ [results truncated to 200 chars]
🔧 Read: src/index.ts
   ↳ (450 lines)
🔧 Edit: src/index.ts
   ↳ Modified successfully
✅ Done — 3 turns, $0.05, 12s
```

Much easier to follow what Claude is doing than scrolling through JSON blobs. A companion script also converts the raw execution output into human-readable markdown for the Step Summary page.

**Fixed `allowed_tools` / `disallowed_tools`.** These inputs were completely non-functional in upstream ([#181](https://github.com/anthropics/claude-code-action/issues/181), [#287](https://github.com/anthropics/claude-code-action/issues/287), [#724](https://github.com/anthropics/claude-code-action/issues/724), [#690](https://github.com/anthropics/claude-code-action/issues/690)). The action accepted them but never passed them through. Now they actually work, including Bash sub-patterns like `Bash(git:*)` that pre-approve command families for unattended CI execution.

**npm fallback for Alpine/musl runners.** The upstream `install.sh` downloads a native binary that requires `posix_getdents`, a function only available in Alpine edge's musl (1.2.5-r21+), not in any stable Alpine release. Since my runner uses `node:alpine3.21`, I added musl detection via `ldd --version`. On musl systems, the action skips the native binary entirely and installs via `npm install -g @anthropic-ai/claude-code`, which runs on Node.js and avoids the incompatibility. On glibc runners (Ubuntu, Debian), the original native install path is unchanged.

That said, I plan to stay close to upstream and keep the fork updated. The goal is minimal divergence, not a full-blown fork with its own roadmap.

## Upstream Sync

This is the part I'm most proud of. Keeping a fork in sync manually is painful and error-prone, so I automated it.

A weekly workflow runs two jobs:

**Job 1: Check upstream** fetches from `anthropics/claude-code-action`, compares commit counts, and creates (or updates) a tracking issue with:
- How many commits behind/ahead
- Latest upstream commit log
- File change statistics
- List of conflict-prone files (the ~8 files with Forgejo platform branches)

**Job 2: Auto-merge** kicks in if there are new upstream commits:
1. Checks for an existing sync PR (reuses it if found)
2. Computes which fork-modified files overlap with upstream changes (conflict risk assessment)
3. Attempts `git merge upstream/main`
4. Hands the result to Claude Code (the action calling itself) with a detailed prompt:
   - Resolve conflicts (fork changes take priority)
   - Review ALL upstream changes for Forgejo compatibility, even in files we haven't modified
   - Check for renamed imports, changed signatures, new GitHub-only assumptions
   - Run tests, typecheck, format
   - Create or update a PR linking back to the tracking issue

The prompt is specific about what Claude should check. Upstream might rename an export that our Forgejo data fetcher imports, or add a new function that assumes GitHub's API shape. Claude reviews the full diff, not just the conflicting lines.

```text
# From the sync workflow prompt:
Even files we haven't modified can break our fork. Check upstream changes for:
- Renamed/moved imports or exports that our Forgejo-specific files depend on
- Changed function signatures, types, or interfaces used by our adapter code
- New assumptions (e.g. GitHub-only APIs) that don't hold on Forgejo
- Removed or restructured code that our platform branches reference
```

The whole thing is self-hosted: the sync workflow runs the Claude Code Action on Forgejo, using the very fork it's syncing. Claude reviews its own upstream updates.

When the sync PR is created, CI kicks in automatically: unit tests, TypeScript type checking, and Prettier formatting run in parallel. On top of that, a self-test workflow runs the action against itself in agent mode on Forgejo. It sends Claude a known prompt ("respond with FORGEJO_PLATFORM_TEST_OK"), then verifies the execution file exists and contains the expected marker. If the upstream merge broke something in the platform detection, token handling, or SDK execution, this catches it before the PR gets merged.

## Status

It works. Tag mode and agent mode both function across issue comments, PR review comments, and workflow_dispatch triggers. I've validated the core scenarios against my Forgejo instance.

The repo lives on my Forgejo instance, but I mirror it to [Codeberg](https://codeberg.org/) for public access. A workflow triggers on every push to `main`, strips instance-specific files (`.forgejo/` workflows, internal PRDs), and force-pushes a clean branch to Codeberg via SSH deploy key.

Is it battle-tested across every edge case? No. But it handles my daily workflow: Claude reviewing PRs, responding to questions on issues, and running automated tasks. That's enough for now.

The [Forgejo setup guide](https://codeberg.org/vtmocanu/claude-code-action/src/branch/main/docs/forgejo.md) on the Codeberg mirror covers prerequisites, workflow templates, bot user setup, known limitations, and troubleshooting.

<span class="kubecon-egg" title="✈ Filed at 30,000ft · en route to KubeCon 2026 · Amsterdam · via Tailscale · with Claude">✈</span>

