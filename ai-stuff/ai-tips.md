# AI Tips

<img src="/images/ai-tips.png" alt="AI Tips" style="max-width: 700px; width: 100%; height: auto;" />

Practical tips for getting real value from AI coding assistants. Takes months to build a useful setup, here's what I've learned.

{{< callout type="info" >}}
**Quick reference:** The [Claude Code Cheat Sheet](https://cc.storyfox.cz/) is a great printable quick-reference for keyboard shortcuts, slash commands, and CLI flags.
{{< /callout >}}

{{< callout type="info" >}}
**Romanian speakers:** If you're just getting started with Claude Code, check out [this introductory video in Romanian](https://www.youtube.com/watch?v=U-OxWYRqFC8).
{{< /callout >}}

## The Three Things That Matter Most

If you take nothing else from this page:

**1. AI doesn't remove the need for discipline—it raises the cost of not having it.**

Spec-driven development, testing, and documentation are more important than ever. When AI can generate code at 10x the pace, the fundamentals that keep a codebase healthy (clear specs up front, tests that actually run, docs that stay honest) become the difference between compounding progress and compounding mess. Skip them and AI accelerates your drift into chaos just as fast as it accelerates feature delivery.

**The beauty of AI agents is that all of this is now simplified: you can have AI write the specs, tests, and docs for you. Your job shifts from writing them to reviewing them.**

Testing is where this bites first and hardest. If AI can't run tests, you're stuck in a ping-pong loop: AI writes code → you run it → it breaks → you paste logs → AI fixes → repeat. That's slow and exhausting.

Set up your environment so AI can:
- Run unit tests directly
- Execute the code and see results
- Check logs and errors itself

And **always include regression/E2E tests**. Without them, AI will fix one thing and break three others. You'll spend more time debugging than if you'd written it yourself. Good test coverage is the guardrail that lets AI move fast without wrecking everything.

**2. Make AI remember what matters**

Use `CLAUDE.md` files (global and per-repo), saved prompts, memory MCPs — whatever it takes. AI that remembers your patterns, conventions, and preferences keeps getting better. AI that starts fresh every time stays mediocre. This is how you build compounding value instead of repeating yourself forever.

**3. Make AI review its own work**

Before calling something done, ask AI to review it like a senior engineer reviewing a PR. Use the pattern: `think like a senior [DOMAIN] engineer and review what we did`. The domain matters — a "senior Go developer" catches different things than a "senior platform engineer" reviewing the same code.

**When to review matters too.** Don't review after every small change — wait until you have a complete feature or finished product. Add feature 1, feature 2, feature 3... then review. With full context of what you've built, AI can suggest meaningful refactors and see architectural improvements that weren't visible when you were building piece by piece.

---

## Sharpen the Saw

<img src="/images/sharpen-the-saw.png" alt="Sharpen the saw" style="max-width: 700px; width: 100%; height: auto;" />

The thing that feels most like procrastination is the thing that compounds fastest. Writing a new prompt, updating `CLAUDE.md`, installing a plugin, wiring up an MCP, refining a skill: it always feels like you should be doing "real" work instead. There's a pressing bug, a feature deadline, a migration everyone's waiting on.

But every hour invested in your setup pays back many times over. A well-tuned harness means AI asks fewer dumb questions, makes fewer dumb mistakes, and needs less hand-holding on every task from that point forward. A neglected harness means explaining the same things to the same model for the 50th time.

**Make it a habit:**

- When you catch yourself repeating instructions, save them to `CLAUDE.md` or a skill
- When an MCP or plugin would have saved you 10 minutes today, install it tomorrow
- When your setup feels crusty, spend a morning cleaning it up instead of pushing through

**This is what makes AI keep getting faster and better.** The engineers who get the most out of AI treat the harness as a first-class artifact, not a side quest. It's the leverage that makes everything else faster.

## Write PRDs, Clear Sessions Often

<img src="/images/write-spec-clear-slate.png" alt="Write the spec, clear the slate" style="max-width: 700px; width: 100%; height: auto;" />

Two practices that used to be context-window workarounds and are still the right default even with 1M tokens.

**Write a PRD before starting anything significant.** Problem, proposed solution, success criteria, a list of milestones. Then point AI at the PRD and work through it one milestone at a time, clearing the session between them. The [PRD workflow on the dot-ai page]({{< ref "dot-ai" >}}#prds-planning-before-implementing) shows how I structure mine.

This originally mattered because context was tight. 200K tokens sounds like a lot until you're ten tool calls into a migration: compaction kicks in, AI hallucinates what it half-remembers, and you spend the rest of the session fighting ghosts. Breaking work into PRD-sized chunks meant each task fit cleanly into a fresh session.

**Does 1M context fix this?** No, and the research is clear:

- **Context rot is measurable.** Chroma benchmarked 18 frontier models; every one degrades as context grows, and the rot starts around 50K tokens, long before you hit the headline limit ([Chroma's context rot research](https://www.trychroma.com/research/context-rot))
- **Lost in the middle.** Information near the start or end of context is recalled reliably; information buried in the middle loses 30%+ accuracy. The more you stuff in, the more you bury
- **Hallucination scales with context.** Opus 4.6 scores 76% on Anthropic's own MRCR retrieval benchmark at 1M. Excellent for a frontier model, and still 24% wrong
- **Costs rise more than caching implies.** Cache reads are 0.1x base input, but cache writes are 1.25x. Every new tool call rewrites part of the cache, and the cached chunk keeps growing, so you pay more on reads too. The "infinite context is free if you cache" intuition is wrong
- **It gets sluggish.** Transformer attention is quadratic; huge contexts slow everything down per token, even ignoring quality

So the practice survives:

- Write the PRD, then execute it in a **fresh** session. This is Anthropic's own advice: ["once the spec is complete, start a fresh session to execute it"](https://claude.com/blog/using-claude-code-session-management-and-1m-context)
- `/clear` often, especially between unrelated tasks
- If you've corrected AI more than twice on the same thing in one session, clear and restart with a better prompt that bakes in what you just learned
- Treat the context window as a workbench you keep tidy, not as your working memory

The PRD file is the memory that survives the clear. That's the whole trick.

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

For a deeper dive on managing permissions across global and per-project Claude Code configs, see [Dippy permissions]({{< ref "dippy-permissions" >}}).

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

## Your Rubber Duck Is Now Alive

<img src="/images/rubber-duck.png" alt="Your rubber duck is now alive" style="max-width: 700px; width: 100%; height: auto;" />

[Rubber duck debugging](https://en.wikipedia.org/wiki/Rubber_duck_debugging) has been around forever: you explain your problem out loud to a rubber duck and halfway through the explanation you figure it out yourself. It works because articulating the problem forces you to think it through.

Now the duck talks back. And it has opinions.

Treat AI as a brainstorming partner, not just a task runner. Before writing code, before filing a plan, pitch the half-baked idea: *"I'm thinking of doing X this way, what do you think? What am I missing?"* You'll often find the idea has a flaw you hadn't spotted, or a simpler alternative you hadn't considered, or a subtle issue that only shows up when you try to explain it out loud.

**When this mode beats delegation:**

- You're weighing two approaches and can't decide which is better
- You have a vague gut feeling something is off but can't articulate it
- You're about to start something big and want a sanity check first
- You just want someone to push back on your reasoning

The duck still does its old job (you think more clearly by explaining yourself), but now it can catch blind spots, propose alternatives, and ask the annoying follow-up questions that expose weak assumptions. It's rubber-duck debugging with teeth.

## Make AI Recommend Best Practices

By default, AI lists options neutrally. Tell it to be opinionated — highlight the best-practice choice and explain why. Add this to your `CLAUDE.md`:

```markdown
- **Prefer best-practice solutions** — when presenting decisions or options,
  always highlight which option is the best-practice approach and why.
```

You want a senior engineer's recommendation, not a menu.

## Cut the Fluff with Output Styles

Claude Code is chatty by default. You ask for a one-line fix and get a preamble, the fix, a summary of what just happened, and a closing offer to help further. It burns tokens, hides the actual answer, and adds up over a session.

The best fix isn't a `CLAUDE.md` rule (those get appended as a user message and don't replace the verbose default). It's a custom **output style**, an official Claude Code feature that directly modifies the system prompt and persists across sessions.

**Create `~/.claude/output-styles/terse.md`:**

```markdown
---
name: Terse
description: Minimal, direct responses. No preamble, no summaries, no fluff.
keep-coding-instructions: true
---

# Terse Mode

- Lead with the answer or action. No preamble, no restating the question.
- No trailing summaries of what you just did. The diff speaks for itself.
- No sycophantic openers ("Great question!", "Sure!", "Absolutely!").
- One sentence beats three. Skip filler words and hedges.
- For investigations: report findings as a bulleted list, not prose.
- Skip "Let me..." / "I'll now..." narration before tool calls.
- No closing offers ("Let me know if you need anything else!").
- User instructions always win. If asked for detail, provide it.
```

<details>
<summary><strong>Full version (what I actually use)</strong></summary>

```markdown
---
name: Terse
description: Minimal, direct responses. No preamble, no summaries, no fluff.
keep-coding-instructions: true
---

# Terse Mode

Respond with the minimum text needed to convey the answer or action. Optimize for the user's reading time, not for thoroughness or politeness.

## Rules

- Lead with the answer or action. No preamble, no restating the question.
- No trailing summaries of what you just did. The diff and tool calls speak for themselves.
- No sycophantic openers ("Great question!", "Sure!", "Absolutely!", "You're right!").
- One sentence beats three. Skip filler words, hedges, and transitional phrases.
- For code changes: show the change, then at most one line of context if non-obvious. Do not narrate the edit.
- For investigations: report findings as a short bulleted list, not prose paragraphs.
- For multi-step work: report milestones, not every intermediate step.
- Only explain reasoning when the user asks "why" or the decision is non-obvious and load-bearing.
- Skip "Let me..." / "I'll now..." narration before tool calls. Just call the tool.
- No closing offers ("Let me know if you need anything else!", "Happy to help with...").
- When asked a yes/no question, the first word should be Yes or No.

## What to keep

- Decisions that need user input.
- Errors, blockers, or surprises that change the plan.
- File paths and line numbers when referencing code (`file.ts:42`).
- Confirmation requests before risky or irreversible actions.

## Override

User instructions always win. If the user explicitly asks for a detailed explanation, walkthrough, or verbose output, provide it.
```

</details>

Activate it with `/config` → Output style → Terse, or set it globally in `~/.claude/settings.json`:

```json
{
  "outputStyle": "Terse"
}
```

You'll need to start a new session for it to take effect (output styles are loaded once at session start so prompt caching stays stable).

**Why output styles beat a `CLAUDE.md` rule:**

- They **replace** parts of the default system prompt instead of being appended after it, so they actually counter the verbose defaults rather than fighting them.
- No per-message input token tax beyond the first cached request.
- Set once globally, applies to every project.
- It's a first-class feature with built-in alternatives (`Default`, `Explanatory`, `Learning`) you can switch between via `/config`.

For the full feature reference, see the [official output styles docs](https://code.claude.com/docs/en/output-styles).

**Alternative approach:** [Caveman](https://github.com/JuliusBrussee/caveman) takes a more radical approach to the same problem. Instead of tuning the system prompt, it compresses Claude's output using a custom encoding that strips articles, prepositions, and filler words. I haven't tried it yet, but it's worth a look if output styles alone aren't aggressive enough for your taste.

## Feed AI Your Hunches

When you notice something (a weird line in the logs, a pattern you half-remember, a suspicion about which file is involved), tell AI immediately. Don't let it spin in circles while you sit there thinking "I bet it's the cache layer" and waiting for it to figure that out on its own.

AI isn't a mind reader. If you saw a flash of something in a dashboard two days ago, or you remember this service had a similar bug last month, or you just have a gut feeling about where to start: say it. Worst case it's wrong and AI corrects you. Best case you save 20 minutes of it re-exploring ground you've already covered.

**The trap:** it's tempting to watch AI flail a bit because it feels like a test. "If it can't figure it out without hints, it's not really smart, right?" Wrong. You have context AI doesn't. Withholding it doesn't prove anything, it just burns time and tokens.

The best collaboration runs both directions at once: you feeding in what you know, AI challenging what you assume.

## Question Everything

AI confidently states things. Some are true, some aren't. **Always challenge it — don't assume it's right just because it sounds right.**

When something feels off (or even when it doesn't), push back:

- **Make it convince you.** "Why do you think that? Walk me through it." If the answer is hand-wavy, the claim probably is too.
- **Ask for links.** Official docs, release notes, source code, GitHub issues. If it can't cite anything, treat the claim as a guess.
- **Do your own research when still unsure.** Read the linked page yourself. Don't trust the summary of the summary.

AI can be wrong for boring reasons: it's trained on data from months or years ago, it's confusing two similar libraries, or it's just being lazy and pattern-matching on what an answer *usually* looks like instead of actually checking.

Worst case you confirm it and move on. Best case you catch a mistake before it ships or learn something new in the process.

## AI Defaults to Stale Versions

AI's training cutoff means it reaches for versions that were current months (or years) ago. Before pinning anything — chart, image tag, runtime, dependency — tell it to look up the latest stable (or LTS) from the project's release page and pin to that. Never floating tags like `latest`.

## Verify Bugs — Make AI Prove It

When AI claims there's a bug in an open source project, don't just take its word for it. Make it **clone the repo and investigate the source code** to confirm. AI can misread docs, hallucinate behavior, or confuse versions. But when you have it trace the actual code path — reading the handler, the frontend component, the RBAC check, the test cases — you get a real answer, not a guess.

I've used this pattern on three real bug reports to date:

- **Flux Operator RBAC bug**: AI traced the missing "Run Job" button through frontend → API → RBAC → tests and found `resource.go` checking workload actions against the wrong API group, with mock data masking it in the test suite. ([issue #677](https://github.com/controlplaneio-fluxcd/flux-operator/issues/677))
- **Renovate Operator onboarding detection**: AI found a naive `strings.Contains("onboarding")` log parser that matched debug messages present in every run, falsely reporting all repos as un-onboarded. ([issue #114](https://github.com/mogenius/renovate-operator/issues/114), shipped in v2.4.1)
- **The bug behind the bug fix**: when the v2.4.1 fix didn't actually solve the problem, AI traced it deeper and found Renovate emits a 190KB+ log line that exceeds `bufio.Scanner`'s 64KB default. ([issue #117](https://github.com/mogenius/renovate-operator/issues/117))

Full entries with status and project links on the [Co-Authored with AI]({{< ref "co-authored-with-ai" >}}) page.

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

{{< callout type="info" >}}
**Deep dive:** [Anatomy of the .claude/ folder](https://blog.dailydoseofds.com/p/anatomy-of-the-claude-folder) covers the full directory structure, from `CLAUDE.md` to custom commands, skills, agents, and permission settings.
{{< /callout >}}

## Split Instructions with `.claude/rules/`

Once your `CLAUDE.md` gets big enough, maintaining one giant file becomes painful. The `.claude/rules/` folder lets you split instructions into separate files by concern. Every `.md` file in there gets loaded automatically alongside `CLAUDE.md`.

**The real power is path-scoped rules.** Add YAML frontmatter to a rule file and it only activates when Claude is working on matching paths:

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "src/handlers/**/*.ts"
---

All handlers must return the `{ data, error }` shape.
Use zod for request body validation.
```

Claude won't load this file when editing a React component. It only kicks in when working inside `src/api/` or `src/handlers/`.

This scales well for teams too. Each team maintains their own domain's rules independently, no merge conflicts in a shared `CLAUDE.md`.

## Take a Break

AI-assisted coding is addictive. You get into a flow where ideas turn into working code faster than ever, one task leads to another, and suddenly hours have vanished. The dopamine loop of "prompt → result → prompt → result" is real.

Don't forget to step away. Stretch, hydrate, touch grass. Your best ideas often come when you're not staring at a terminal.

Not convinced it's a real problem? Read [I take my laptop to the gym so Claude doesn't have downtime](https://www.claudecodecamp.com/p/i-take-my-laptop-to-the-gym-so-claude-doesn-t-have-downtime). That's the edge of the cliff.

{{< youtube ZJEnQOsMtsU >}}

