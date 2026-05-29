# Context Is the New Code

{{< callout type="info" >}}
**TL;DR.** Patrick Debois (the person who coined "DevOps", now DevRel at Tessl) argues that AI coding agents live or die on the quality of their context, not the model. He proposes a Context Development Lifecycle (generate, evaluate, distribute, observe) so context earns the version control, review, testing and observability that code already has. The talk is 27 minutes and says it all, so watch it; my notes below are just the hooks.
{{< /callout >}}

<div style="margin-top: 1.5rem;"></div>

{{< youtube bSG9wUYaHWU >}}

*"Context Is the New Code" by Patrick Debois (Tessl), AI Engineer conference, May 2026.*

## Why it stuck with me

- **The thesis.** Code has version control, review, testing, CI/CD, observability. Context (prompts, specs, skills, instructions) has none of that, yet. That gap is the next discipline.
- **Context Development Lifecycle (CDLC):** generate → evaluate → distribute → observe, cycling continuously. Treat your markdown instructions as a first-class asset and productivity compounds; copy-paste prompts into a repo and hope, and you hit a ceiling fast.
- **Context as shareable assets.** Package context, install it across projects and teams, discover it via a registry. The packaging unit he lands on is the [skill]({{< ref "ai-stuff/commands-vs-mcp-vs-skills" >}}): not a lone markdown file, but a bundle of scripts, docs and context.
- **Brutally honest.** On the current state of shared skills: "99.9%, and I mean that in a very sincere way, of the skills is crap."
- **"Context debt."** A community follow-up (Jarosław Wasowski) coined this as the parallel to technical debt. The name fits.

## Further reading

- [The Context Development Lifecycle](https://tessl.io/blog/context-development-lifecycle-better-context-for-ai-coding-agents/) (Tessl blog)
- [Two Weeks After "Context Is the New Code"](https://jedi.be/blog/2026/two-weeks-after-context-is-the-new-code/) (Patrick's own follow-up: 60k+ views in ten days, translated into four languages, community-extended CDLC variants)

