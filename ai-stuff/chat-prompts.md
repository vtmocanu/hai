# Chat Prompts

## The Problem

AI confidently produces output — code, documentation, configs — but doesn't automatically verify its own work. It writes a Helm chart and moves on. It documents a workflow and assumes it's correct. Without explicit prompts to review, you get first-draft quality that might have bugs, inaccuracies, or missed edge cases.

The fix is simple: **ask AI to review its own work before you accept it.**

These are quick one-liners I use mid-conversation to catch issues before they become problems.

---

## Verify What We Wrote

```
do some research online and think like a senior devops engineer,
does it make sense what we wrote? did we lie about anything?
```

**When to use:** After writing documentation, decision records, or technical explanations.

**What you get:** AI fact-checks claims, verifies technical accuracy, and flags anything that sounds wrong. It might find outdated information, incorrect assumptions, or statements that need nuance.

---

## Self-Review Like a Senior Engineer

```
think like a senior [DOMAIN] engineer and review what we did
```

Replace `[DOMAIN]` with the relevant expertise:

| Domain | What it catches |
|--------|-----------------|
| `DevOps` | Infrastructure anti-patterns, CI/CD issues, deployment risks |
| `Go developer` | Code style, error handling, idioms, missing tests |
| `platform engineer` | Architecture issues, security gaps, scalability concerns |
| `UX engineer` | Flow problems, clarity issues, user experience friction |

**When to use:** After completing a feature or significant piece of work. Don't review after every small change — wait until you have something complete.

**What you get:** A second opinion that catches things you missed. Different domains surface different issues — a "senior Go developer" finds different problems than a "senior security engineer" reviewing the same code.

---

## PR-Style Code Review

```
you're a senior Go developer, review the changes in our branch vs main
like you're reviewing a PR — check for bugs, style issues, missing tests
```

**When to use:** Before merging, or when you want a thorough review of accumulated changes.

**What you get:** Structured feedback like a real PR review — bugs, style issues, missing tests, potential improvements. AI compares your branch against main and reviews the diff, not just individual files.

---

## The Pattern

These prompts share a common structure:

1. **Assign a role** — "think like a senior X engineer"
2. **Define the task** — "review what we did" / "fact-check this"
3. **Set expectations** — "like you're reviewing a PR"

The role matters. A generic "review this" gets generic feedback. A specific "senior platform engineer reviewing for production readiness" gets focused, relevant critique.

---

**See also:** [Saved Prompts]({{< ref "saved-prompts" >}}) for longer, reusable prompts that persist across sessions.

