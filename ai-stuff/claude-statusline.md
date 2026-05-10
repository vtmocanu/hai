# Custom Claude Code Status Line

Claude Code has a fully customizable status line. You point it at a shell script, it pipes in session data as JSON, and your script renders whatever you want. I've been iterating on mine for a while; the current version is **v2.1.6** and now lives in a public repo: [codeberg.org/vtmocanu/cc-statusline](https://codeberg.org/vtmocanu/cc-statusline).

The v2 layout uses two lines with diagonal corner cuts and width-synchronized lines. Each session gets a unique color from a 12-color palette (hashed from the session ID), so when I have multiple sessions open, I can tell them apart at a glance.

![Claude Code statusline v2](/images/claude-statusline-v2.png)

**Line 1** (project-colored background): session topic, folder, git branch + status, Kubernetes context

**Line 2** (black background): model + effort level, elapsed time, context window bar, 5h/7d API quota bars with reset times, Claude service status icon

All meters are color-coded: green under 50%, gold 50-80%, coral 80%+.

{{< tabs >}}
{{< tab name="Installation" icon="lightning-bolt" >}}

## Install

The statusline lives in a public repo on Codeberg: [vtmocanu/cc-statusline](https://codeberg.org/vtmocanu/cc-statusline). Clone it and run the installer:

```bash
git clone https://codeberg.org/vtmocanu/cc-statusline.git
cd cc-statusline
./install.sh
```

The installer extracts the chosen ref via `git archive` (so it never mutates your working tree), copies the scripts into `~/.local/share/cc-statusline/`, and prints the JSON snippets to paste into `~/.claude/settings.json`.

To pin a specific release:

```bash
./install.sh --version v2.1.6
```

To uninstall:

```bash
./install.sh --uninstall
```

## Configure Claude Code

After running the installer, paste the snippets it prints into `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.local/share/cc-statusline/statusline.sh"
  },
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/.local/share/cc-statusline/hooks/session-topic-capture.sh"
          }
        ]
      }
    ]
  }
}
```

The `UserPromptSubmit` hook is **optional**: it's the bit that calls Claude Haiku to generate the per-session "Project: Focus" topic label. If you don't want that, just leave the hook out and the statusline will skip the topic block.

Restart Claude Code. The topic appears after your first prompt (it runs async, so it shows on the second render).

## Requirements

- macOS or Linux, `bash` 4+, `jq`, `perl`, `curl`
- `gsed` on macOS (`brew install gnu-sed`), used by the optional session-topic hook
- A [Nerd Font](https://www.nerdfonts.com/) v3+ in your terminal for the powerline corners and icons
- Claude Code v2.1.80+ for native rate-limit data
- `kubectl` is optional (used to display the current context)

The installer checks for these and tells you what's missing.

## Updating

```bash
cd /path/to/cc-statusline
git pull
./install.sh
```

The installer detects the previous install and reports the upgrade transition (`upgraded v2.1.1 -> v2.1.6`).

## Source, issues, contributions

Everything is on Codeberg: [vtmocanu/cc-statusline](https://codeberg.org/vtmocanu/cc-statusline). The repo has a CI pipeline (shellcheck + a JSON-fixture test harness), a `CHANGELOG.md`, and a `KNOWN_ISSUES.md` for limitations. Issues and pull requests welcome.

{{< /tab >}}
{{< tab name="Technical Deep Dive" icon="code" >}}

## How It Works

Claude Code calls your script after each assistant message, piping a JSON blob to stdin with session metadata (`model`, `cwd`, `context_window` with token breakdown, `cost`, `session_id`, `rate_limits`, etc.). Your script reads it and prints ANSI-colored output to stdout.

## Key Features

**Per-session color** - The session ID is hashed to one of 12 colors (blue, green, pink, amber, teal, purple, sky, olive, coral, steel, khaki, violet). You can pin a specific color to a project via `~/.claude/statusline-color-overrides.json`:

```json
{ "/path/to/your/project": 4 }
```

**Session topic** - A `UserPromptSubmit` hook calls Claude Haiku with a snippet of your conversation and writes a "Project: Focus" label to `~/.claude/session-topics/{session_id}.txt`. The statusline reads this file on each render. Rate-limited to prompt 1 and every 10 prompts after that.

**Native API quota** - Since [v2.1.80](https://github.com/anthropics/claude-code/releases/tag/v2.1.80), Claude Code passes a `rate_limits` field directly in the statusline JSON input, with `five_hour` and `seven_day` windows containing `used_percentage` (integer) and `resets_at` (Unix epoch). No more manual API calls, token management, or caching needed.

**Thinking effort level** - The effort level isn't exposed in the statusline JSON, so the script detects it by parsing the session transcript for `/model` and `/effort` command outputs. The grep is anchored to the JSON `"content":"<local-command-stdout>` prefix to avoid matching conversation text that discusses effort levels. Falls back to the `effortLevel` setting in `~/.claude/settings.json`, then defaults to `medium`. Inspired by [ccstatusline](https://github.com/sirmalloc/ccstatusline)'s approach.

**Claude service status** - A separate fetch script (`~/.claude/claude-status-fetch.sh`) calls the [status.claude.com](https://status.claude.com) Statuspage JSON API and writes a one-line cache to `/tmp/claude-service-status`. The statusline checks the cache age on each render and spawns a background refresh if it's older than 60 seconds. The status is displayed as a single icon (`✓`/`⚠`/`~`/`✗`) to save horizontal space for rate limit detail.

**Terminal tab title** - The tab title is set to the session topic (or folder name if no topic exists yet).

**Adaptive line width** - Both lines are padded to match the wider one. `SAFE_WIDTH` (default 110, override with `STATUSLINE_WIDTH` env var) controls how much rate limit detail is generated on line 2 (full bars + reset times, compact bars, or percentages only). Line 1 is also width-enforced: if it exceeds `SAFE_WIDTH`, components are progressively truncated (K8S context first, then branch name, then topic) to prevent `cli-truncate` from silently dropping line 2.

**Crash safety** - The script uses `set -uo pipefail` (no `-e`) and a `trap 'printf "\n"' EXIT` to guarantee at least empty output on crash. This prevents the statusline from silently disappearing when external commands (`git`, `kubectl`, `jq`) return non-zero. Stdin is read with `timeout 2` to avoid blocking forever if Claude Code fails to pipe JSON.

## Rendering Gotchas

Claude Code's statusline renderer has some undocumented behaviors I discovered through trial and error:

- **`cli-truncate` drops lines** - Claude Code uses Ink's `Text wrap="truncate"` internally, which calls `cli-truncate` on the entire multi-line output. If line 1 exceeds the container's available width, all subsequent lines are silently dropped. This is why both lines must stay under a safe width limit.
- **No terminal control sequences** - `\033[K` (erase to end of line), `\033[?7l/h` (wrap control), and `\033[${COLS}G` (cursor positioning) cause rendering glitches. Stick to standard ANSI SGR color codes only (`\033[...m`).
- **`tput cols` returns 80 in pipes** - Claude Code pipes JSON to the script's stdin, so `tput cols` returns the pipe default (80), not the actual terminal width. The `COLUMNS` env var is also unset. Use a fixed safe width instead of detecting terminal size.
- **Process group cleanup** - Claude Code kills the entire process group when the statusline process exits. Background subshells (`(cmd) &`) don't survive, making async cache updates impossible.
- **Use `printf '%b'` over `echo -e`** - More reliable escape handling, recommended by the official docs. Prepend `\033[0m` to each line to override Claude Code's default dim styling.
- **jq float rounding** - `jq`'s `floor` can return values like `14.000000000000002`. Truncate with `${PCT%%.*}` before using in bash arithmetic.
- **`set -e` kills silently** - With `set -e`, any external command returning non-zero (git in a non-repo dir, kubectl with no context, jq on malformed JSON) terminates the script with zero output. Claude Code sees nothing and the statusline disappears. Use `set -uo pipefail` without `-e` instead.
- **Bash IFS tab collapsing** - Do NOT use `@tsv` with `IFS=$'\t' read`. Bash treats tab as "IFS whitespace," meaning consecutive tabs (from empty fields like `agent.name=""`) are collapsed into a single delimiter. This silently shifts all subsequent variables. Use jq's `@sh` format with `eval` instead: each field becomes a properly quoted `VAR='value'` assignment, sidestepping IFS entirely.

## Credits

The v2 design is based on [lee-fuhr's statusline gist](https://gist.github.com/lee-fuhr/68141b3ad716a96950cd111c749442b6), adapted with my own icon preferences and added Kubernetes context.

{{< /tab >}}
{{< /tabs >}}

## Adapting It

Each segment in `statusline.sh` is independent. Fork the repo, edit the bits you want, and run your own copy via `bash /path/to/your/fork/statusline.sh` in `settings.json`. If you build something interesting on top of it, [open an issue or PR](https://codeberg.org/vtmocanu/cc-statusline) — I'd love to see it.

