# Context Is the New Code

{{< callout type="info" >}}
**TL;DR.** Had this on my watchlist since it dropped. Finally sat down with it, and it's good, really good. Patrick Debois (yes, the one who defined the whole DevOps movement) makes the case that context is the new code: your prompts, specs and skills need version control, review, testing and observability, same as code. Basically CI/CD for AI, and a real piece of where AI platform engineering is heading. And he is already building it. The talk says it all, so watch it; my notes below are just the hooks.
{{< /callout >}}

<div style="margin-top: 1.5rem;"></div>

{{< youtube bSG9wUYaHWU >}}

*"Context Is the New Code" by Patrick Debois (Tessl), AI Engineer conference, May 2026.*

## Why it stuck with me

- **The thesis lands hard.** Code has version control, review, testing, CI/CD, observability. Context (prompts, specs, skills, instructions) has none of that, yet. That gap is the next discipline, and naming it that plainly is half the value of the talk.
- **There's a lifecycle for it.** The Context Development Lifecycle (CDLC): generate → evaluate → distribute → observe, cycling continuously. Treat your markdown instructions as a first-class asset and productivity compounds; copy-paste prompts into a repo and hope, and you hit a ceiling fast.
- **Context wants to be shared.** Package it, install it across projects and teams, discover it via a registry. The unit he lands on is the [skill]({{< ref "ai-stuff/commands-vs-mcp-vs-skills" >}}): not a lone markdown file, but a bundle of scripts, docs and context.
- **He doesn't sugarcoat it.** On the current state of shared skills: "99.9%, and I mean that in a very sincere way, of the skills is crap." Hard to argue.
- **"Context debt" is the keeper.** A community follow-up (Jarosław Wasowski) coined it as the parallel to technical debt. Once you hear it, you can't unsee it.

## Further reading

- [The Context Development Lifecycle](https://tessl.io/blog/context-development-lifecycle-better-context-for-ai-coding-agents/) (Tessl blog)
- [Two Weeks After "Context Is the New Code"](https://jedi.be/blog/2026/two-weeks-after-context-is-the-new-code/) (Patrick's own follow-up: 60k+ views in ten days, translated into four languages, community-extended CDLC variants)

