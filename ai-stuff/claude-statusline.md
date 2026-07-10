# Custom Claude Code Status Line

Claude Code has a fully customizable status line. You point it at a shell script, it pipes in session data as JSON, and your script renders whatever you want. I've been iterating on mine for a while; the current version is **v2.3.0** and now lives in a public repo: [github.com/vtmocanu/cc-statusline](https://github.com/vtmocanu/cc-statusline).

{{< callout type="info" >}}
**TL;DR**: Claude Code's default status line tells me almost nothing about a session, and with several sessions open at once I lose track of which is which. So I built a two-line ANSI status bar that hashes each session to its own color and packs in the things I actually watch: the session topic, git and Kubernetes context, context-window usage, prompt-cache hit rate, and live 5h/7d API quota bars. The result is a glanceable dashboard per session, installable from a public repo with one script.
{{< /callout >}}

The v2 layout uses two lines with diagonal corner cuts and width-synchronized lines. Each session gets a unique color from a 12-color palette (hashed from the session ID), so when I have multiple sessions open, I can tell them apart at a glance.

![Claude Code statusline v2](/images/claude-statusline-v2.png)

**Line 1** (project-colored background): session topic, folder, git branch + status, Kubernetes context

**Line 2** (black background): model + effort level, optional account profile badge (the `MM` chip above, useful when juggling multiple Claude Code logins), elapsed time, context window bar, prompt-cache hit rate (`⚡ 99%` above), 5h/7d API quota bars with reset times and optional pace arrows, Claude service status icon

All meters are color-coded: green under 50%, gold 50-80%, coral 80%+ (the cache hit-rate meter inverts this, since a high cache rate is good).

{{< tabs >}}
{{< tab name="Installation" icon="lightning-bolt" >}}

## Install

The statusline lives in a public repo on GitHub: [vtmocanu/cc-statusline](https://github.com/vtmocanu/cc-statusline). Since v2.8.0 the easiest way in is Homebrew:

```bash
brew install vtmocanu/tap/cc-statusline
```

That puts a `cc-statusline` command on your PATH, pulls in the dependencies, and prints the `settings.json` snippets to paste (re-read them anytime with `brew info cc-statusline`).

No Homebrew? The installer works everywhere `git` does:

```bash
git clone https://github.com/vtmocanu/cc-statusline.git
cd cc-statusline
./install.sh
```

It extracts the chosen ref via `git archive` (so it never mutates your working tree), copies the scripts into `~/.local/share/cc-statusline/`, and prints the same snippets. `./install.sh --version v2.8.0` pins a release, `./install.sh --uninstall` removes it.

## Configure Claude Code

Paste the snippets into `~/.claude/settings.json`. With the brew install:

```json
{
  "statusLine": {
    "type": "command",
    "command": "cc-statusline",
    "refreshInterval": 60
  },
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "/opt/homebrew/opt/cc-statusline/libexec/hooks/session-topic-capture.sh"
          }
        ]
      }
    ]
  }
}
```

(With `install.sh`, the command is `bash ~/.local/share/cc-statusline/statusline.sh` and the hook path lives under `~/.local/share/` instead.)

`refreshInterval` is optional but worth having: Claude Code normally re-runs the statusline only on activity, so an idle session shows stale reset times and service health. With it set, the statusline also re-renders every 60 seconds.

The `UserPromptSubmit` hook is **optional**: it's the bit that calls Claude Haiku to generate the per-session "Project: Focus" topic label. If you don't want that, just leave the hook out and the statusline will skip the topic block.

Restart Claude Code. The topic appears after your first prompt (it runs async, so it shows on the second render).

## Requirements

- macOS or Linux, `bash`, `jq`, `perl`, `curl`, GNU `timeout` (coreutils; not stock on macOS)
- A [Nerd Font](https://www.nerdfonts.com/) v3+ in your terminal for the powerline corners and icons
- Claude Code v2.1.80+ for native rate-limit data
- `kubectl` is optional (used to display the current context)

The brew formula declares all of these; the installer checks for them and tells you what's missing.

## Updating

```bash
brew update && brew upgrade cc-statusline
```

Or with the installer:

```bash
cd /path/to/cc-statusline
git pull
./install.sh
```

The installer detects the previous install and reports the upgrade transition (`upgraded v2.2.1 -> v2.3.0`).

## Hacking on it

The brew wrapper has a built-in dev switch: point it at a working tree and the next render runs your clone instead of the brewed copy, no `settings.json` edits, even in already-running sessions.

```bash
mkdir -p ~/.config/cc-statusline
echo ~/src/cc-statusline > ~/.config/cc-statusline/dev-dir   # enter dev mode
rm ~/.config/cc-statusline/dev-dir                           # back to the brewed copy
```

## Source, issues, contributions

Everything is on GitHub: [vtmocanu/cc-statusline](https://github.com/vtmocanu/cc-statusline). The repo has a CI pipeline (shellcheck + a JSON-fixture test harness), a `CHANGELOG.md`, and a `KNOWN_ISSUES.md` for limitations. Issues and pull requests welcome.

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

**Native API quota** - Since [v2.1.80](https://github.com/anthropics/claude-code/releases/tag/v2.1.80), Claude Code passes a `rate_limits` field directly in the statusline JSON input, with `five_hour` and `seven_day` windows containing `used_percentage` (a percentage that can be fractional, so the script truncates it to a whole number) and `resets_at` (Unix epoch). No more manual API calls, token management, or caching needed.

**Prompt-cache hit rate** - Line 2 shows a compact `⚡ NN%` right after the context size: the share of the last API call's input tokens served from the prompt cache (`cache_read_input_tokens / (input_tokens + cache_creation_input_tokens + cache_read_input_tokens)`, from `context_window.current_usage`). The color scale is inverted from the other meters, green when most of the context is cached (cheap), coral when it's cold, so a healthy session reads green. It's hidden before the first API call and right after a `/compact` (when `current_usage` is null), and `STATUSLINE_CACHE=0` turns it off.

**Rate-limit pace arrows** - Each quota can show an arrow after its percentage, projecting where usage is headed by the time the window resets (`projected = used% × window_seconds / elapsed_seconds`, plain bash integer math, no `bc`): `↑` in coral when you're on track to blow through the cap before it resets, `→` in gold when you're roughly on pace. Under-consuming, the common case, shows no arrow at all, so the glyph reads as an alert rather than constant decoration. Suppressed during the first 2% of a window (too little signal yet) and toggleable with `STATUSLINE_PACE=0`.

**Thinking effort level** - The effort level isn't exposed in the statusline JSON, so the script detects it by parsing the session transcript for `/model` and `/effort` command outputs. The grep is anchored to the JSON `"content":"<local-command-stdout>` prefix to avoid matching conversation text that discusses effort levels. Falls back to the `effortLevel` setting in `~/.claude/settings.json`, then defaults to `medium`. Inspired by [ccstatusline](https://github.com/sirmalloc/ccstatusline)'s approach.

**Claude service status** - A separate fetch script (`~/.claude/claude-status-fetch.sh`) calls the [status.claude.com](https://status.claude.com) Statuspage JSON API and writes a one-line cache to `/tmp/claude-service-status`. The statusline checks the cache age on each render and spawns a background refresh if it's older than 60 seconds. The status is displayed as a single icon (`✓`/`⚠`/`~`/`✗`) to save horizontal space for rate limit detail.

**Terminal tab title** - The tab title is set to the session topic (or folder name if no topic exists yet).

**Account profile badge (opt-in)** - For people who switch between multiple Claude Code logins (work vs. personal), the statusline can show a colored badge on line 2 identifying the active account. It reads `.oauthAccount.accountUuid` directly from `~/.claude.json` (Claude Code's own state file, also swapped atomically by [claude-account-switcher](https://github.com/Symbioose/claude-account-switcher)), looks up the UUID in `~/.claude/profile-labels.json`, and renders the configured label in the configured color. No network call, no Keychain access; it works the same on macOS and Linux. Disabled when the mapping file is absent, when its top-level `enabled` is `false`, or when `STATUSLINE_PROFILE=0` is set. Unknown UUIDs render as a `XXXXXX?` hint in gray so you know to add a mapping (helper script at `~/.claude/label-current-profile.sh <label> [color]` pulls the UUID + email + org from `~/.claude.json` and writes the entry).

**Adaptive line width** - Both lines are padded to match the wider one. `SAFE_WIDTH` (default 110, override with `STATUSLINE_WIDTH` env var) controls how much rate limit detail is generated on line 2 (full bars + reset times, compact bars, or percentages only). Line 1 is also width-enforced: if it exceeds `SAFE_WIDTH`, components are progressively truncated (K8S context first, then branch name, then topic) to prevent `cli-truncate` from silently dropping line 2. Rate-limit detail takes priority over the cache readout: the quota tier is chosen from the base width *excluding* cache, so the reset countdowns are never squeezed out, and the `⚡ NN%` meter renders only in whatever width is left over. Bump `STATUSLINE_WIDTH` (I run 130) to fit both comfortably.

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

Each segment in `statusline.sh` is independent. Fork the repo, edit the bits you want, and run your own copy via `bash /path/to/your/fork/statusline.sh` in `settings.json`. If you build something interesting on top of it, [open an issue or PR](https://github.com/vtmocanu/cc-statusline); I'd love to see it.

