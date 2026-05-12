# AI radar

A running log of AI-adjacent links I want to remember: tools to try, projects to watch, articles I read or mean to. Some are tested and I have an opinion. Most are on the *want-to-try* pile.

Status legend:

- ✅ tried it, recommend
- 🟡 tried it, mixed feelings
- ❌ tried it, would not bother again
- 👀 on my list, not tried yet

{{< filterable-table column="Status" valueLabels=`{"✅":"Tried, recommend","🟡":"Tried, mixed","❌":"Tried, would skip","👀":"Not tried yet"}` >}}
| Date added | Item | What it is | Status |
|------------|------|------------|--------|
| 2026-05-12 | [dot-ai](https://github.com/vfarcic/dot-ai) ([my write-up](/ai-stuff/dot-ai/)) | Viktor Farcic's AI-powered MCP server for Kubernetes operations. I use it for shared/git skills — custom prompts synced from a central git repo (my `wxs/ai-resources`) and exposed as `/dot-ai-*` slash commands globally across all Claude Code projects. | ✅ Tested, recommend |
| 2026-05-12 | [Infisical agent-vault](https://github.com/Infisical/agent-vault) | Open-source credential broker that securely proxies API calls for AI agents by injecting credentials at the network layer — agents never see the secrets directly. | 👀 Not tried yet — waiting on [#124](https://github.com/Infisical/agent-vault/issues/124) |
| 2026-05-12 | [multica.ai](https://www.multica.ai/) | Open-source platform that turns coding agents into real teammates — assign tasks to AI agents, track progress, and manage a combined human-agent workforce from a centralized dashboard. | 👀 Not tried yet |
| 2026-05-12 | [clawd-on-desk](https://github.com/rullerzhou-afk/clawd-on-desk) | Pixel desktop pet that reacts in real-time to AI coding agents (Claude Code, Codex, Cursor, Gemini CLI, …) with 12 animated states. Permission-bubble UI for tool approvals, eye tracking in idle, session dashboard for concurrent agents. | ✅ Tested, recommend |
| 2026-05-12 | [agnix](https://github.com/agent-sh/agnix) | Linter and LSP for AI coding assistants — validates `CLAUDE.md`, `AGENTS.md`, `SKILL.md`, hooks and MCP configs with autofixes. Rust CLI with plugins for VS Code and other major IDEs. | 👀 Not tried yet — waiting on [#909](https://github.com/agent-sh/agnix/issues/909) for per-file rule overrides |
| 2026-05-12 | [nanoclaw](https://github.com/nanocoai/nanoclaw) ([my deployment: Vasile](/ai-stuff/vasile-nanoclaw-bot/)) | Lightweight container-isolated personal AI agent framework. Connects to WhatsApp, Telegram, Slack, Discord, Gmail; memory, scheduled jobs, runs on Anthropic's Agents SDK. Built from scratch, ~15 source files. | ✅ Tested, recommend |
| 2026-05-12 | [Dippy](https://github.com/ldayton/Dippy) ([my write-up](/ai-stuff/dippy-permissions/)) | CLI tool that runs as a Claude Code `PreToolUse` hook to manage permissions. Supports wildcards at any position, guidance messages on prompt, layered global + project configs — things `settings.json` still can't do natively. | ✅ Tested, recommend |
| 2026-05-12 | [RTK](https://github.com/rtk-ai/rtk) ([my write-up](/ai-stuff/rtk/)) | Rust CLI proxy that sits between Claude Code and shell commands, filtering noisy tool output (passing tests, ANSI escapes, repeated log lines) before it reaches the context window. | 🟡 Tried, hit some issues, might revisit |
| 2026-05-12 | [ccxray](https://github.com/lis186/ccxray) | Transparent HTTP proxy + dashboard that gives "X-ray vision" into Claude Code sessions — inspect every request/response, tool call, and token spend in real time. | 🟡 Interesting, couldn't find a use case |
{{< /filterable-table >}}

