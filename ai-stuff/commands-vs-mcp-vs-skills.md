# Commands vs MCP vs Skills

AI coding agents like Claude Code can be extended in three main ways: **commands**, **MCP tools**, and **skills**. Each serves a different purpose and has different trade-offs.

Viktor Farcic explains this better than I could, so watch the video first, then use the summary below as a quick reference.

{{< youtube xAIN7YHXfCY >}}

## Quick Reference

### Commands (Slash Commands)

Pre-defined **prompt templates** stored as markdown files, triggered explicitly by the user (e.g., `/commit`, `/review`).

- Simple to create and maintain (just markdown files)
- Consistent execution every time
- Must be invoked manually; the LLM won't discover them on its own
- No standard location across agents (Claude Code uses `.claude/commands/`, Cursor uses `.cursor/rules/`)

### MCP Tools

Server-based tools exposing external capabilities through a **standardized protocol**. The LLM decides when to call them based on context.

- Extends what the agent can *do* (query APIs, databases, monitoring systems)
- Standardized interface that works across all MCP-compatible agents
- Invoked automatically; the LLM picks the right tool for the job
- Higher implementation cost, network latency, and security considerations

For more on MCP, see [What is MCP?]({{< ref "mcp-intro" >}}).

### Skills

Markdown-based workflows that the LLM can invoke **both explicitly and automatically** when the context matches. Only skill descriptions load into the system context initially; the full instructions inject when triggered.

- Dual invocation: explicit (`/skill-name`) or automatic based on context
- Minimal context usage (descriptions only until triggered)
- Teaches the agent *how* to use its abilities well
- Distribution across projects is still a challenge

## When to Use What

| Mechanism | Best for | Invocation |
|-----------|----------|------------|
| **Commands** | Explicit workflows you always trigger manually | User only |
| **MCP** | External integrations (APIs, databases, tools) | LLM decides |
| **Skills** | Complex workflows that benefit from automatic detection | Both |

The key insight: **MCP gives agents abilities. Skills teach agents how to use those abilities well.** Commands are for when you want full control over when something runs.

## Source

- [Full transcript](https://devopstoolkit.live/ai/commands-vs-mcp-vs-skills-what-i-use/) on DevOps Toolkit

