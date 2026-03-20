# CLAUDE.md

`CLAUDE.md` is how you give Claude Code project-specific memory. It's a markdown file that Claude reads automatically at the start of every session — think of it as persistent instructions that survive between conversations.

## Where CLAUDE.md Files Live

| Location | Scope | Example Use |
|----------|-------|-------------|
| `~/.claude/CLAUDE.md` | All projects (global) | Tool preferences, CLI aliases, workflow rules |
| `./CLAUDE.md` (project root) | This project only | Architecture, commands, conventions |

Both are read together. Keep them lean — they consume context window on every session.

## What Goes In Mine

### PRD Workflow

I use [dot-ai]({{< ref "dot-ai" >}}) for PRD creation and management (`/dot-ai:prd-create`, `/dot-ai:prd-next`, `/dot-ai:prd-done`). After completing a PRD, Claude automatically analyzes the milestone dependency graph and structures them into parallel phases — so I can fire multiple agents on independent milestones simultaneously.

I also isolate PRD development with branches or worktrees. On `/prd-start`, Claude asks me which one I want:

- **Branch** — just creates a feature branch and switches to it. Simple, stays in the same directory.
- **Worktree** — creates a worktree at `../{REPO_NAME}-prd-{N}` (e.g., `../wdb-prd-50`) and gives me a copy-paste command to cd into it and start a new Claude session. Claude Code [can't cd to sibling directories](https://github.com/anthropics/claude-code/issues/2841) for security reasons, so you need to launch a fresh session from the worktree.

On `/prd-done`, Claude auto-detects whether I'm on a branch or worktree (via `git worktree list`) and cleans up accordingly. This keeps the main checkout clean and lets multiple agents work on different PRDs in parallel without conflicts.

These instructions live in my global `CLAUDE.md`:

```markdown
## PRD Workflow: Parallel Milestone Analysis

After completing any PRD, analyze the milestone dependency graph
to identify parallelization opportunities:

1. Map dependencies: which milestones depend on others
2. Group into phases: independent milestones run as parallel agents
3. Present the execution plan with phases, dependencies, files, and repos
4. Structure PRD milestones using Phase 1 (parallel) / Phase 2 (sequential) format

### PRD Branch/Worktree Workflow

On `/prd-start`, ask the user: branch or worktree?

- Branch: `git checkout -b feature/prd-50-some-feature main`
- Worktree: `git worktree add -b feature/prd-50-some-feature ../wdb-prd-50 main`
  Then tell the user: `cd ../wdb-prd-50 && claude`

On `/prd-done`, auto-detect and clean up:
- Worktree (detect via `git worktree list`): `git worktree remove ../wdb-prd-50`
- Branch (detect via current branch): `git checkout main`
```

### Documentation Auto-Update Triggers

When I say "in the future we should..." or "next time...", Claude immediately updates the relevant documentation without me having to ask. This turns operational learnings into persistent knowledge automatically.

### Skill Learning

When a session uses a `dot-ai-*` skill and encounters corrections, edge cases, or workarounds, Claude suggests running `/reflect` to capture learnings and update the skill. This closes the feedback loop: use a skill, discover something new, improve the skill for next time.

```markdown
- **Skill learning**: After completing a task that used a `dot-ai-*` skill and
  encountered corrections, edge cases, or workarounds, suggest running
  `/dot-ai-reflect` to capture learnings and update the skill.
```

Combined with the "in the future" trigger, this means knowledge is captured automatically whether it's about a skill or a general workflow.

### Platform-Specific Quirks

My global `CLAUDE.md` includes things like:
- Use `gsed` not `sed` (macOS)
- Use `/bin/ls` not `ls` (aliased to `eza` which hangs in Claude Code)
- Use `fd` and `rg` over `find` and `grep`

Tool-specific knowledge (like Forgejo CLI commands, Crossplane patterns, etc.) lives in skills with self-describing trigger conditions rather than in `CLAUDE.md`. This keeps the global file lean and lets skills auto-load only when needed.

### Code Quality Self-Review

I have Claude review its own work before presenting it — like a senior engineer reviewing a junior's PR:

```markdown
## Code Quality

Before finishing any task, review your own work as a senior engineer would review a junior's PR:
- Is this the simplest correct approach, or did I take a shortcut?
- Did I miss edge cases, race conditions, or error paths?
- Are there security implications (injection, path traversal, unvalidated input, leaked secrets)?
- Would I approve this in a code review, or would I send it back?

If something feels wrong, fix it before presenting. Flag security risks explicitly
— never silently hope the user will catch them.
```

This catches issues before they reach me. I use this in [nanoclaw](https://nanoclaw.net/) as part of the agent's core instructions.

### Per-Repo Instructions

Each project's `CLAUDE.md` covers what matters for that repo. My Flux monorepo's is the largest — it includes:
- Key directory structure and navigation
- Flux CLI commands for reconciliation and debugging
- Drift detection workflows and common fixes
- S3 backup system (s3bkp) architecture and annotations
- Same-color restore runbooks

## CLAUDE.md vs Saved Prompts

These serve different purposes:

| | CLAUDE.md | [Saved Prompts]({{< ref "saved-prompts" >}}) |
|---|-----------|---------------|
| **Loaded** | Automatically, every session | On-demand when invoked |
| **Context cost** | Always consumed | Only when needed |
| **Best for** | Small, always-relevant rules | Large domain expertise |
| **Examples** | "Use gsed not sed" | Full Crossplane/KCL guide |

Rule of thumb: if it's under ~20 lines and always relevant, put it in `CLAUDE.md`. If it's a deep knowledge base, make it a [saved prompt]({{< ref "saved-prompts" >}}) or [dot-ai prompt]({{< ref "dot-ai" >}}).

## Tips

- **Start small** — add rules as you discover patterns, not upfront
- **Let Claude write it** — after a session, say "add what we learned to CLAUDE.md"
- **Review periodically** — outdated instructions waste context and cause confusion
- **Use the "in the future" trigger** — say "in the future, always check X before Y" and Claude updates docs immediately

