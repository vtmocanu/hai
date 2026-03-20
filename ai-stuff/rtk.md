# RTK: Cutting Token Waste in Claude Code

{{< callout type="info" >}}
**Update (2026-03-14):** There's an [open issue (#582)](https://github.com/rtk-ai/rtk/issues/582) reporting that RTK's hook can increase costs by 18% in some scenarios. The compressed output causes Claude to generate more output tokens to compensate for missing information, outweighing the input savings. Worth monitoring before adopting.
{{< /callout >}}

Every time Claude Code runs a shell command (`ls`, `git diff`, `kubectl get`, `grep`), the raw output goes straight into the context window. That includes passing test lines nobody reads, verbose directory listings, whitespace noise in diffs, and full stack traces when only the error message matters.

These tokens don't just cost once. They get re-read on every subsequent turn of the conversation. A noisy `pytest` run with 200 passing tests and 1 failure? That full output sits in context for the rest of the session, getting counted again with every message.

The compounding is significant. A few hundred wasted tokens per command, across dozens of commands, re-read across multiple turns: suddenly you're burning through your context budget on noise instead of actual work.

## What RTK Does

[RTK](https://github.com/rtk-ai/rtk) (Rust Token Killer) is a single-binary CLI proxy written in Rust. It sits between Claude Code and your shell commands, intercepting tool output and stripping the noise before it reaches the context window. No changes to how you work. Claude Code doesn't even know it's there.

It uses four compression strategies:

1. **Smart filtering** - strips boilerplate, passing tests, progress bars, ANSI noise
2. **Grouping** - aggregates similar items (files by directory, errors by category)
3. **Truncation** - preserves relevant context while cutting redundancy
4. **Deduplication** - collapses repeated log lines with occurrence counts

The tool supports 30+ commands out of the box: `ls`, `find`, `grep`, `git` operations, `kubectl`, `docker`, test runners (`pytest`, `cargo test`, `go test`, `npm test`), linters, and package managers.

## How It Hooks Into Claude Code

RTK registers itself as a [PreToolUse hook](https://docs.anthropic.com/en/docs/claude-code/hooks) in Claude Code. When Claude runs a Bash command, the hook rewrites it: `git status` becomes `rtk git status`. Claude only sees the filtered output. The rewrite is transparent.

This is the right approach. Subagents and background processes may ignore `CLAUDE.md` instructions, but hooks are always enforced at the tool execution level.

## Setup

This is probably the easiest tool setup I've ever done. Two commands, 30 seconds:

```bash
brew install rtk
rtk init -g --hook-only
```

Done. No config files to edit, no environment variables, no YAML to write. It just works. The `--hook-only` flag is important: without it, `rtk init` also adds an `RTK.md` reference to your `CLAUDE.md`. That file is ~400 tokens that get loaded into every conversation, which defeats the purpose of a token-saving tool.

What `rtk init -g` does under the hood: it installs a shell script at `~/.claude/hooks/rtk-rewrite.sh` and registers a `PreToolUse` hook in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "$HOME/.claude/hooks/rtk-rewrite.sh"
      }]
    }]
  }
}
```

## Real Savings

Here's my `rtk gain` output after 956 commands:

```
RTK Token Savings (Global Scope)
════════════════════════════════════════════════════════════

Total commands:    956
Input tokens:      9.4M
Output tokens:     4.4M
Tokens saved:      4.9M (52.5%)
Total exec time:   99m9s (avg 6.2s)

By Command
─────────────────────────────────────────────────────────────
  #  Command          Count  Saved    Avg%    Time   Impact
─────────────────────────────────────────────────────────────
 1.  curl                 2   3.2M   99.8%    73ms  ██████████
 2.  read                39   1.4M   16.5%     6ms  ████░░░░░░
 3.  diff                 5  49.4K   97.9%    89ms  ░░░░░░░░░░
 4.  grep                44  44.5K   22.5%    31ms  ░░░░░░░░░░
 5.  find                59  36.6K   77.8%    42ms  ░░░░░░░░░░
 6.  ls                 186  33.7K   69.6%     6ms  ░░░░░░░░░░
 7.  go test              1  28.7K  100.0%    3.4s  ░░░░░░░░░░
```

The big wins are on noisy commands: `curl` responses (99.8% stripped), `diff` output (97.9%), `find` results (77.8%), and `ls` listings (69.6%). File reads save less (16.5%) because most of the content is actually relevant, but the volume is high so it still adds up.

The practical impact: fewer context compactions, longer productive sessions, and less time re-explaining context that got compressed away.

## Useful Commands

| Command | What it does |
|---------|-------------|
| `rtk gain` | Token savings summary |
| `rtk gain --history` | Recent command history with per-command savings |
| `rtk gain --graph` | ASCII graph of 30-day trends |
| `rtk gain --daily` | Day-by-day breakdown |
| `rtk discover` | Analyzes past Claude Code sessions and finds commands that weren't routed through RTK |
| `rtk proxy <cmd>` | Runs a command through RTK without filtering (for debugging) |

`rtk discover` is particularly useful after initial setup. Here's what it found scanning my last 30 days of Claude Code sessions:

```
Scanned: 295 sessions, 9347 Bash commands
Already using RTK: 4 commands (0%)

MISSED SAVINGS -- Commands RTK already handles
────────────────────────────────────────────────────────────
Command              Count    Est. Savings
kubectl get           2057    ~356.3K tokens
git add                995    ~148.8K tokens
grep -r                472    ~65.1K tokens
find                   314    ~61.6K tokens
gh api                 164    ~49.0K tokens
helm show              266    ~31.5K tokens
head -10               130    ~28.2K tokens
curl -s                243    ~22.8K tokens
ls -la                  98    ~8.4K tokens
────────────────────────────────────────────────────────────
Total: 4788 commands -> ~779.8K tokens saveable
```

Nearly **780K tokens** I could have saved, just from commands RTK already knows how to filter. The top offender is `kubectl get` with over 2,000 calls. That's a lot of verbose table output sitting in context windows for no reason.

It also shows unhandled commands (things RTK doesn't filter yet, like `flux get` or `flux-local build`), which you can open as issues on the GitHub repo.

## How It Handles Failures

When a command fails, you want the full output for debugging, not a compressed summary. RTK handles this with a "tee" feature: on failure, it saves the unfiltered output to `~/.local/share/rtk/tee/` so the LLM can access the complete error context without re-executing the command.

## Worth It?

The overhead is <10ms per command. Setup is 30 seconds. There's nothing to configure beyond the initial `rtk init`.
For anyone using Claude Code regularly, especially on the Pro plan where context budget matters, it's an easy win. The tokens you save on tool output noise translate directly into longer sessions and fewer compaction cycles.

