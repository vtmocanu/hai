# AI Tips

Practical tips for getting real value from AI coding assistants. Takes months to build a useful setup — here's what I've learned.

{{< callout type="info" >}}
**Romanian speakers:** If you're just getting started with Claude Code, check out [this introductory video in Romanian](https://www.youtube.com/watch?v=U-OxWYRqFC8).
{{< /callout >}}

## The Three Things That Matter Most

If you take nothing else from this page:

**1. Make AI remember what matters**

Use `CLAUDE.md` files (global and per-repo), saved prompts, memory MCPs — whatever it takes. AI that remembers your patterns, conventions, and preferences keeps getting better. AI that starts fresh every time stays mediocre. This is how you build compounding value instead of repeating yourself forever.

**2. AI must test its own work**

This is non-negotiable. If AI can't run tests, you're stuck in a ping-pong loop: AI writes code → you run it → it breaks → you paste logs → AI fixes → repeat. That's slow and exhausting.

Set up your environment so AI can:
- Run unit tests directly
- Execute the code and see results
- Check logs and errors itself

And **always include regression/E2E tests**. Without them, AI will fix one thing and break three others. You'll spend more time debugging than if you'd written it yourself. Good test coverage is the guardrail that lets AI move fast without wrecking everything.

**3. Make AI review its own work**

Before calling something done, ask AI to review it like a senior engineer reviewing a PR. Use the pattern: `think like a senior [DOMAIN] engineer and review what we did`. The domain matters — a "senior Go developer" catches different things than a "senior platform engineer" reviewing the same code.

**When to review matters too.** Don't review after every small change — wait until you have a complete feature or finished product. Add feature 1, feature 2, feature 3... then review. With full context of what you've built, AI can suggest meaningful refactors and see architectural improvements that weren't visible when you were building piece by piece.

---

## Use AI to Generate Prompts

Use AI to generate prompts for AI — inception, I know :)

**My prompt:**

<div class="wrap-code">

```
update prompt to say when we create a new page to generate some tags, check existing tags, see which match and ask the user if proposed tags are ok
```

</div>

**Resulting prompt:**

<div class="wrap-code">

```markdown
### Tags

When creating a new page:

1. **Check existing tags** by searching `content/` for `tags:` patterns
2. **Reuse existing tags** where they match (consistency over creativity)
3. **Propose new tags** only when needed
4. **Ask the user** to confirm the proposed tags before adding them

Keep tags lowercase, short, and meaningful. Avoid generic tags like "blog", "update", "milestone".
```

</div>

## Pre-Allow Read-Only Tools

Allow as many read-only tools as possible in your Claude Code settings (`~/.claude/settings.json`). This lets Claude explore and investigate freely without constantly asking for permission — it only prompts for write operations.

**Example settings:**

```json
{
  "permissions": {
    "allow": [
      "Bash(ls*)",
      "Bash(cat*)",
      "Bash(head*)",
      "Bash(tail*)",
      "Bash(grep*)",
      "Bash(rg*)",
      "Bash(find*)",
      "Bash(fd*)",
      "Bash(git status*)",
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git branch*)",
      "Read",
      "Glob",
      "Grep",
      "WebFetch",
      "WebSearch"
    ]
  }
}
```

The result: Claude can read files, search code, check git state, and browse documentation without nagging you. It only asks permission when it needs to write, edit, or execute something potentially destructive.

## Set Effort Level to Max

Claude Code has an `effortLevel` setting that controls how much thinking the model does before responding. Set it to `max` in your `~/.claude/settings.json`:

```json
{
  "effortLevel": "max"
}
```

Higher effort means deeper reasoning, better code, and fewer mistakes. The tradeoff is slower responses and higher token usage, but for real development work it's worth it.

You can also change it per-session with `/model`, but setting it globally means you never forget.

For a detailed breakdown of effort levels and which settings make sense per subscription tier, see [this guide](https://llmx.tech/blog/how-to-change-claude-code-effort-level-best-settings-per-subscription-tier/).

## Be Strategic with MCPs

[MCP servers]({{< ref "mcp-intro" >}}) consume context window — each tool definition takes tokens. Having too many MCPs globally means Claude starts every conversation with less room for actual work.

**Strategy:**

- **Global MCPs** (`~/.claude/settings.json`): Only essential, always-needed tools (e.g.: Context7 for docs, [dot-ai]({{< ref "dot-ai" >}}) for k8s/shared prompts, prometheus)
- **Per-repo MCPs** (`.claude/settings.json` in repo): Project-specific tools (e.g.: homeassitant, harbor)

This keeps context lean for simple tasks while having full tooling available where needed.

{{< callout type="info" >}}
**Recommended watch:** [MCP Servers Explained: Why Most Are Useless (And How to Fix It)](https://www.youtube.com/watch?v=7baGJ1bC9zE) — good breakdown of what makes an MCP actually useful vs just noise.
{{< /callout >}}

## Tell AI to Work in Parallel

For migrations and repetitive tasks, tell AI to process multiple items in parallel. Instead of "migrate this resource" → review → "now this one" → review, say "migrate 5 resources in parallel."

**Why 5?** It's fast enough to feel efficient, but still few enough to review each one properly. This works for tasks where each item needs independent review — not for fully scripted migrations where you'd review the whole batch at once anyway.

**Example prompt:**
```
migrate the next 5 harbor replications in parallel,
show me the changes for each so I can review them together
```

## Spin Up Expert Review Panels

When you need a thorough review, don't just ask AI to "review this." Instead, spin up multiple agents in parallel, each with a different senior expert persona. They read the same document but focus on completely different concerns.

**Example prompt:**

```
spin up 3 agents in parallel:
one senior devops, one senior harbor expert, one python senior developer
to review this PRD
```

Each agent gets the full document plus a role-specific review brief. The DevOps engineer focuses on operational risk, rollback plans, and cross-repo coordination. The domain expert (Harbor, in this case) focuses on API edge cases, protocol-specific gotchas, and things only someone deep in that ecosystem would catch. The developer focuses on architecture, testing gaps, and code quality.

**Why this works better than a single review:**

- **Different lenses see different things.** The DevOps reviewer caught that a file path default would break when code moved between repos. The Harbor expert caught that proxy cache projects would be flagged as false positives. The Python developer flagged that a 4,000-line merge had zero testing plan (the only RED rating across all three reviews).
- **Domain experts don't overlap much.** In practice, the three reviews had very little redundancy. Each surfaced unique findings the others missed entirely.
- **It's fast.** All three run in parallel, so you get three thorough reviews in the time it takes for one.

**Tips:**

- Pick roles that cover different angles: ops, domain, code quality. Three generic "senior engineers" would overlap too much.
- Give each agent a specific review brief with questions tailored to their expertise, not just "review this."
- After the reviews come back, synthesize the findings yourself and update the document. The agents find the gaps; you decide which ones matter.

## Run Multiple Sessions — The Dialup Workaround

AI agents can feel like dialup. You ask something, wait while it thinks, reviews, runs tests... and you're just sitting there. The workaround? **Run multiple Claude Code sessions in parallel.**

I typically have 3 terminal tabs open, each with its own Claude Code session working on a different task. While one agent is deep in a migration, I'm reviewing another's output, and a third is exploring a codebase for me. When one finishes and needs my input, I switch to it, give feedback, and switch back.

**How it works in practice:**
- **Tab 1:** Working on a PRD milestone — writing code, running tests
- **Tab 2:** Investigating a bug in a different repo
- **Tab 3:** Generating documentation or doing a migration

Each session is independent, so they don't interfere. Use different repos or [worktrees](https://git-scm.com/docs/git-worktree) if working on the same repo to avoid conflicts.

**The tradeoff:** More cognitive load from context-switching between conversations. You're essentially managing 3 junior developers at once — checking in, reviewing, course-correcting. But it beats staring at a spinner. The total throughput is much higher even if each individual session gets slightly less attention.

**Tip:** Start with 2 sessions until the switching feels natural, then scale to 3. Beyond 3, the context-switching cost usually outweighs the parallelism benefit.

## You're the Tech Lead

Treat AI like a capable junior you're coaching to senior. You're the team lead, architect, QA, and product owner rolled into one.

**The uncomfortable truth:** AI is dumb. Same as people — without context, it makes bad assumptions. Your job is to train it, coach it, help it grow from junior to senior. Think of it as a team member whose experience you store in context files instead of their brain.

**This takes time.** Expect a few months before you have a truly useful setup. You're building muscle memory together — yours for prompting effectively, AI's (via context) for understanding your world.

**What this means in practice:**

- **Always review the work** — don't blindly accept output. Read it, understand it, question it.
- **Make it test its own work** — ask for verification, have it run the tests, check the logs.
- **Challenge assumptions** — AI might be wrong (you catch a mistake) or right (you learn something). Either way, you win.
- **Train your preferences** — use global/local `CLAUDE.md`, saved prompts, memory MCPs. The more context AI has about how you work, the less you repeat yourself.
- **Provide feedback** — when something's off, say so. "That's too verbose", "We don't do it that way", "Simplify this". It adjusts.
- **Learn from failures together** — when you find the right way to do something, update the context immediately. Make AI learn from mistakes so they don't repeat.

The goal isn't to micromanage — it's to maintain ownership. You're responsible for what ships, not the AI.

## Make AI Recommend Best Practices

By default, AI lists options neutrally. Tell it to be opinionated — highlight the best-practice choice and explain why. Add this to your `CLAUDE.md`:

```markdown
- **Prefer best-practice solutions** — when presenting decisions or options,
  always highlight which option is the best-practice approach and why.
```

You want a senior engineer's recommendation, not a menu.

## Question Everything

AI confidently states things. Some are true, some aren't. Get in the habit of asking "wait, is that right?" — worst case you confirm it, best case you learn something or catch a mistake.

## Verify Bugs — Make AI Prove It

When AI claims there's a bug in an open source project, don't just take its word for it. Make it **clone the repo and investigate the source code** to confirm. AI can misread docs, hallucinate behavior, or confuse versions. But when you have it trace the actual code path — reading the handler, the frontend component, the RBAC check, the test cases — you get a real answer, not a guess.

**Example 1 — Flux Operator RBAC bug:** AI told me the Flux Operator web UI's "Run Job" button wasn't showing due to a bug. Instead of just filing an issue based on speculation, I had it clone the repo and trace the full flow: frontend component → API response → RBAC check → test coverage. It found that `resource.go` checks workload actions (`restart`) against the wrong API group, and that mock data was hiding the bug in tests. That level of evidence turned a "maybe this is broken" into a [confirmed, well-documented bug report](https://github.com/controlplaneio-fluxcd/flux-operator/issues/677).

**Example 2 — Renovate Operator onboarding detection:** The Renovate Operator UI showed "No Config (renovate not onboarded)" for all repositories despite them being fully onboarded. AI traced the log parser code path, found it used a naive `strings.Contains("onboarding")` check that matched debug messages like `"checkOnboarding()"` present in every run, and [filed a detailed bug report](https://github.com/mogenius/renovate-operator/issues/114) with the exact code location and suggested fix. The maintainers shipped a fix in the next release (v2.4.1).

**Example 3 — The bug behind the bug fix:** After the v2.4.1 fix landed, onboarding detection *still* didn't work. Instead of assuming the fix was wrong, AI cloned the repo, read the new parser code, and verified the logic was correct. Then it measured the actual Renovate log output — found that Renovate emits a `"packageFiles with updates"` line that can be 190KB+, while Go's `bufio.Scanner` silently stops at 64KB. The parser never reaches the `"Repository finished"` line at the end of the logs. A [Go reproducer script confirmed the hypothesis](https://github.com/mogenius/renovate-operator/issues/117), and a smaller repo (with logs under 64KB) worked perfectly — proving the diagnosis. One-line fix: `scanner.Buffer(make([]byte, 0), 1024*1024)`.

**The pattern:**
```
clone the repo to /tmp and investigate further,
make sure this is actually a bug before we report it
```

This also works in reverse — sometimes the investigation reveals you were holding it wrong, saving you from filing an embarrassing false bug report.

## Run Bash Commands Inline with `!`

You can run shell commands directly from inside Claude Code by prefixing them with `!`. No need to exit the session, open another terminal, or ask Claude to run it for you.

```
> ! kubectl get pods -A
> ! git log --oneline -5
> ! cat /tmp/output.json | jq '.items[0]'
```

Useful for quick checks while Claude is working, verifying something yourself before Claude continues, or running commands you don't need Claude to see or interpret.

## Manage Memories with `/memory`

Claude Code has two memory systems: `CLAUDE.md` files (you write these, they're instructions and rules) and **auto memory** (Claude writes these, things it learns as you work together).

The `/memory` command opens an interactive menu where you can:
- View all loaded memory and `CLAUDE.md` files
- Toggle auto memory on/off for the current project
- Open memory files for editing

Auto memory is stored at `~/.claude/projects/<project>/memory/` and contains a `MEMORY.md` index file plus optional topic files (e.g., `debugging.md`, `patterns.md`). Claude automatically saves useful patterns, corrections, and preferences here across sessions.

```
> /memory
```

You can also ask Claude directly to remember something: "remember that we always use pnpm in this project" and it saves it to auto memory. Or just edit `CLAUDE.md` yourself for things you want to enforce as rules rather than suggestions.

## Take a Break

AI-assisted coding is addictive. You get into a flow where ideas turn into working code faster than ever, one task leads to another, and suddenly hours have vanished. The dopamine loop of "prompt → result → prompt → result" is real.

Don't forget to step away. Stretch, hydrate, touch grass. Your best ideas often come when you're not staring at a terminal.

{{< youtube ZJEnQOsMtsU >}}

