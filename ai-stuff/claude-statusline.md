# Custom Claude Code Status Line

Claude Code has a fully customizable status line. You point it at a shell script, it pipes in session data as JSON, and your script renders whatever you want. I've gone through two versions so far.

## v2: Per-Session Colors + Session Topics

The current version uses a two-line layout with diagonal corner cuts. Each session gets a unique color from a 12-color palette (hashed from the session ID), so when I have multiple sessions open, I can tell them apart at a glance.

![Claude Code statusline v2](/images/claude-statusline-v2.png)

**Line 1** (project-colored background):

- **Session topic** (prompts: Session parsing) - LLM-generated "Project: Focus" label, updated every 10 prompts by calling Claude Haiku in the background
- **Folder** (wxs/prompts) - current repo as parent/folder
- **Git branch** (main) - with staged/modified/untracked counts
- **Kubernetes context** (k8s-blue-cc) - active cluster

**Line 2** (black background):

- **Model + effort** (Opus 4.6 max) - active Claude model with current [thinking effort level](https://llmx.tech/blog/how-to-change-claude-code-effort-level-best-settings-per-subscription-tier/), color-coded (sage for max/high, gray for medium, coral for low)
- **Elapsed time** (3h45m) - color-coded green/gold/red as a proxy for context degradation
- **Context window** (6% of 1000k) - visual bar + percentage
- **5h usage** (3%, resets in 4h31m) - rolling 5-hour API quota bar
- **7d usage** (5%, resets in 5d17h) - rolling 7-day API quota bar
- **Claude service status** - live indicator from [status.claude.com](https://status.claude.com), auto-refreshed every 60 seconds in the background. Shows `✓ ok` (green) when operational, `⚠ incident title` (coral) during incidents, `✗` (red) for major outages

All meters are color-coded: green under 50%, gold 50-80%, coral 80%+.

### Key Features

**Per-session color** - The session ID is hashed to one of 12 colors (blue, green, pink, amber, teal, purple, sky, olive, coral, steel, khaki, violet). You can pin a specific color to a project via `~/.claude/statusline-color-overrides.json`:

```json
{ "/path/to/your/project": 4 }
```

**Session topic** - A `UserPromptSubmit` hook calls Claude Haiku with a snippet of your conversation and writes a "Project: Focus" label to `~/.claude/session-topics/{session_id}.txt`. The statusline reads this file on each render. Rate-limited to prompt 1 and every 10 prompts after that.

**Native API quota** - Since [v2.1.80](https://github.com/anthropics/claude-code/releases/tag/v2.1.80), Claude Code passes a `rate_limits` field directly in the statusline JSON input, with `five_hour` and `seven_day` windows containing `used_percentage` (integer) and `resets_at` (Unix epoch). No more manual API calls, token management, or caching needed.

**Thinking effort level** - The effort level isn't exposed in the statusline JSON, so the script detects it by parsing the session transcript for `/model` and `/effort` command outputs. The grep is anchored to the JSON `"content":"<local-command-stdout>` prefix to avoid matching conversation text that discusses effort levels. Falls back to the `effortLevel` setting in `~/.claude/settings.json`, then defaults to `medium`. Inspired by [ccstatusline](https://github.com/sirmalloc/ccstatusline)'s approach.

**Claude service status** - A separate fetch script (`~/.claude/claude-status-fetch.sh`) calls the [status.claude.com](https://status.claude.com) Statuspage JSON API and writes a one-line cache to `/tmp/claude-service-status`. The statusline checks the cache age on each render and spawns a background refresh if it's older than 60 seconds, so it never blocks rendering. No external scheduler (launchd/cron) needed.

**Terminal tab title** - The tab title is set to the session topic (or folder name if no topic exists yet).

### Installation

{{< callout type="info" >}}
The fastest way to get this running: open Claude Code and paste this page's URL with "implement this statusline for me". It will read the scripts, save them, configure `settings.json`, and set up the hook.
{{< /callout >}}

**Requirements:** [Nerd Font](https://www.nerdfonts.com/) v3+ (for powerline corners and icons), `jq`, `kubectl` (optional). Claude Code v2.1.80+ for native rate limit data.

1. Save both scripts below and make them executable:

```bash
chmod +x ~/.claude/statusline.sh
chmod +x ~/.claude/hooks/session-topic-capture.sh
mkdir -p ~/.claude/session-topics
```

2. Add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline.sh"
  },
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/.claude/hooks/session-topic-capture.sh"
          }
        ]
      }
    ]
  }
}
```

3. Restart Claude Code. The topic will appear after your first prompt (it runs async, so it shows on the second render).

### Full Scripts

{{% details title="statusline.sh (v2)" closed="true" %}}

```bash
#!/usr/bin/env bash
set -euo pipefail

DATA=$(cat)

# ── Extract fields ──────────────────────────────────────────────────────────
IFS=$'\t' read -r MODEL MODEL_ID DIR PCT CTX_SIZE DURATION_MS AGENT MODE < <(
    echo "$DATA" | jq -r '[
        (.model.display_name // "Claude"),
        (try (.model.id // "unknown") catch "unknown"),
        (.cwd // "~" | split("/") | .[-2:] | join("/")),
        (try (
            if (.context_window.remaining_percentage // null) != null then
                100 - (.context_window.remaining_percentage | floor)
            elif (.context_window.context_window_size // 0) > 0 then
                (((.context_window.current_usage.input_tokens // 0) +
                  (.context_window.current_usage.cache_creation_input_tokens // 0) +
                  (.context_window.current_usage.cache_read_input_tokens // 0)) * 100 /
                 .context_window.context_window_size) | floor
            else 0 end
        ) catch 0),
        (.context_window.context_window_size // 200000),
        (.cost.total_duration_ms // 0),
        (.agent.name // ""),
        (.mode // "")
    ] | @tsv'
)
TRANSCRIPT_PATH=$(echo "$DATA" | jq -r '.transcript_path // ""' 2>/dev/null)
CTX_SIZE_K=$((CTX_SIZE / 1000))
COLS=$(tput cols 2>/dev/null || echo 120)

TOPIC=""  # populated after SESSION_ID is extracted below

# ── Effort level detection (transcript → settings → default) ──────────────
EFFORT=""
if [ -n "$TRANSCRIPT_PATH" ] && [ -f "$TRANSCRIPT_PATH" ]; then
    # Match both "/model" and "/effort" outputs, anchored to JSON content field
    # to avoid matching conversation text that mentions these patterns
    EFFORT=$(grep -E '"content":"<local-command-stdout>(Set model to.*effort|Set effort level to)' "$TRANSCRIPT_PATH" 2>/dev/null \
        | tail -1 | grep -oE '\b(low|medium|high|max)\b' | tail -1 || true)
fi
if [ -z "$EFFORT" ]; then
    EFFORT=$(jq -r '.effortLevel // empty' "$HOME/.claude/settings.json" 2>/dev/null || true)
fi
EFFORT=${EFFORT:-medium}

# ── Nerd Font icons ───────────────────────────────────────────────────────
NF_GIT=$'\xee\x82\xa0'       # U+E0A0 powerline branch
NF_FOLDER="󰉋"               # nf-md-folder
NF_MODEL="󰚩"                # nf-md-robot
NF_K8S="󱃾"                  # nf-md-kubernetes
NF_CLOCK=$'\xef\x80\x97'     # U+F017 clock
NF_CORNER_TL=$'\xee\x82\xba'    # U+E0BA top-left corner cut
NF_CORNER_BL=$'\xee\x82\xbe'    # U+E0BE bottom-left corner cut
NF_CORNER_TR=$'\xee\x82\xb8'    # U+E0B8 top-right corner cut
NF_CORNER_BR=$'\xee\x82\xbc'    # U+E0BC bottom-right corner cut

# ── Project-colored background (hash session ID -> unique hue) ────────────
RST="\033[0m"
CWD_FULL=$(echo "$DATA" | jq -r '.cwd // "~"')
PROJECT_ROOT=$(git -C "$CWD_FULL" rev-parse --show-toplevel 2>/dev/null || echo "$CWD_FULL")
SESSION_ID=$(echo "$DATA" | jq -r '.session_id // empty' 2>/dev/null)
PHASH=$(printf '%s' "${SESSION_ID:-$CWD_FULL}" | cksum | cut -d' ' -f1)

# ── Session topic ─────────────────────────────────────────────────────────
if [ -n "${SESSION_ID:-}" ]; then
    TOPIC_FILE="$HOME/.claude/session-topics/${SESSION_ID}.txt"
    [ -f "$TOPIC_FILE" ] && TOPIC=$(cat "$TOPIC_FILE" 2>/dev/null | tr -d '\n' | cut -c1-40)
fi

# Check for manual color override
COLOR_OVERRIDES="$HOME/.claude/statusline-color-overrides.json"
if [ -f "$COLOR_OVERRIDES" ]; then
    COLOR_IDX=$(jq -r --arg p "$PROJECT_ROOT" '.[$p] // empty' "$COLOR_OVERRIDES" 2>/dev/null)
fi
COLOR_IDX=${COLOR_IDX:-$((PHASH % 12))}

case $COLOR_IDX in
    0)  BG_R=105; BG_G=145; BG_B=225 ;;  # blue
    1)  BG_R=130; BG_G=190; BG_B=130 ;;  # green
    2)  BG_R=190; BG_G=130; BG_B=175 ;;  # pink
    3)  BG_R=200; BG_G=170; BG_B=100 ;;  # amber
    4)  BG_R=100; BG_G=185; BG_B=185 ;;  # teal
    5)  BG_R=175; BG_G=130; BG_B=190 ;;  # purple
    6)  BG_R=110; BG_G=170; BG_B=210 ;;  # sky
    7)  BG_R=180; BG_G=190; BG_B=110 ;;  # olive
    8)  BG_R=200; BG_G=140; BG_B=130 ;;  # coral
    9)  BG_R=130; BG_G=170; BG_B=180 ;;  # steel
    10) BG_R=190; BG_G=175; BG_B=120 ;;  # khaki
    11) BG_R=160; BG_G=130; BG_B=190 ;;  # violet
    *)  BG_R=105; BG_G=145; BG_B=225 ;;  # fallback: blue
esac

# Line 1 colors (derived from project palette)
SEP_R=$((BG_R * 40 / 100)); SEP_G=$((BG_G * 40 / 100)); SEP_B=$((BG_B * 40 / 100))
TXT_R=$((BG_R * 15 / 100)); TXT_G=$((BG_G * 15 / 100)); TXT_B=$((BG_B * 15 / 100))

BG1="\033[48;2;${BG_R};${BG_G};${BG_B}m"
B="${RST}${BG1}"
SEP="\033[38;2;${SEP_R};${SEP_G};${SEP_B}m│"
TXT_FG="\033[38;2;${TXT_R};${TXT_G};${TXT_B}m"
TXT_BOLD="\033[38;2;${TXT_R};${TXT_G};${TXT_B};1m"
PROJ_FG="\033[38;2;${BG_R};${BG_G};${BG_B}m"

# ── Line 2 colors (black fill, light gray text, colored % numbers) ──────────
BG2="\033[48;2;0;0;0m"
B2="${RST}${BG2}"
L2_TXT="\033[38;2;170;170;170m"   # light gray
L2_DIM="\033[38;2;80;80;80m"      # dim gray for separators + resets

pct_txt_color() {
    local p=$1
    if   [ "$p" -gt 80 ]; then printf "\033[38;2;225;150;150m"   # coral
    elif [ "$p" -gt 50 ]; then printf "\033[38;2;215;195;125m"   # gold
    else                        printf "\033[38;2;150;210;150m"   # sage
    fi
}

# ── Git info ────────────────────────────────────────────────────────────────
BRANCH=$(git -c core.useBuiltinFSMonitor=false branch --show-current 2>/dev/null || echo "")
GIT_STATUS=""
if [ -n "$BRANCH" ]; then
  STAGED=$(git diff --cached --numstat 2>/dev/null | wc -l | tr -d " ")
  MODIFIED=$(git diff --numstat 2>/dev/null | wc -l | tr -d " ")
  UNTRACKED=$(git ls-files --others --exclude-standard 2>/dev/null | wc -l | tr -d " ")
  [ "$STAGED" -gt 0 ]    && GIT_STATUS="+${STAGED}"
  [ "$MODIFIED" -gt 0 ]  && GIT_STATUS="${GIT_STATUS:+$GIT_STATUS }!${MODIFIED}"
  [ "$UNTRACKED" -gt 0 ] && GIT_STATUS="${GIT_STATUS:+$GIT_STATUS }?${UNTRACKED}"
fi

# ── Kubernetes context ──────────────────────────────────────────────────────
K8S_CTX=$(kubectl config current-context 2>/dev/null || echo "")

# ── Session duration ────────────────────────────────────────────────────────
TOTAL_SEC=$((DURATION_MS / 1000))
H=$((TOTAL_SEC / 3600))
M=$(((TOTAL_SEC % 3600) / 60))
S=$((TOTAL_SEC % 60))
if   [ "$H" -gt 0 ]; then TIME="${H}h${M}m"
elif [ "$M" -gt 0 ]; then TIME="${M}m${S}s"
else TIME="${S}s"
fi

# Color-code elapsed time
if   [ "$H" -gt 2 ]; then TIME_CLR="\033[38;2;225;150;150m"   # coral: 3h+
elif [ "$H" -gt 0 ]; then TIME_CLR="\033[38;2;215;195;125m"   # gold: 1-3h
else                      TIME_CLR="\033[38;2;150;210;150m"   # sage: <1h
fi

# ── Bar builder ─────────────────────────────────────────────────────────────
make_bar() {
    local pct=$1 width=$2 fill_clr="$3" empty_clr="$4" bar=""
    local filled=$((pct * width / 100))
    [ "$pct" -gt 0 ] && [ "$filled" -eq 0 ] && filled=1
    [ "$filled" -gt "$width" ] && filled=$width
    local empty=$((width - filled))
    for ((i=0; i<filled; i++)); do bar+="${fill_clr}▰"; done
    for ((i=0; i<empty; i++));  do bar+="${empty_clr}▱"; done
    printf "%b" "$bar"
}

# ── Rate limit reset formatter (takes Unix epoch) ─────────────────────────
format_reset() {
    local epoch="$1"
    [ -z "$epoch" ] || [ "$epoch" = "null" ] || [ "$epoch" = "0" ] && return
    local now diff
    now=$(date +%s)
    diff=$((epoch - now))
    [ "$diff" -le 0 ] && { printf "now"; return; }
    [ "$diff" -lt 60 ] && { printf "<1m"; return; }
    local d=$((diff / 86400)) h=$(((diff % 86400) / 3600)) m=$(((diff % 3600) / 60))
    if   [ "$d" -gt 0 ]; then printf "%dd%dh" "$d" "$h"
    elif [ "$h" -gt 0 ]; then printf "%dh%dm" "$h" "$m"
    else printf "%dm" "$m"
    fi
}

# ── Count visible columns ──────────────────────────────────────────────────
count_cols() {
    local ESC=$'\033'
    printf "%b" "$1" | sed "s/${ESC}\[[0-9;]*m//g" | tr -d '\n' | LC_ALL=en_US.UTF-8 wc -m | tr -d ' '
}

# ── Line 1 content ─────────────────────────────────────────────────────────
L1C="${RST}${PROJ_FG}${NF_CORNER_TL}${BG1}"
[ -n "$TOPIC" ] && L1C+=" ${TXT_BOLD}${TOPIC}${B} ${SEP}${B}"
L1C+=" ${TXT_FG}${NF_FOLDER} ${DIR} ${B}"
if [ -n "$BRANCH" ]; then
    L1C+="${SEP}${B} ${TXT_FG}${NF_GIT} ${BRANCH}${B}"
    [ -n "$GIT_STATUS" ] && L1C+=" ${TXT_FG}${GIT_STATUS}${B}"
fi
[ -n "$AGENT" ] && L1C+=" ${TXT_FG}${AGENT}${B}"
[ -n "$MODE" ]  && L1C+=" ${SEP}${B} \033[38;2;150;100;0;1m${MODE}${B}"
if [ -n "$K8S_CTX" ]; then
    L1C+=" ${SEP}${B} ${TXT_FG}${NF_K8S} ${K8S_CTX}${B}"
fi
L1C+=" "

# ── Line 2 content ──────────────────────────────────────────────────────────
CTX_CLR=$(pct_txt_color "$PCT")
CTX_BAR=$(make_bar "$PCT" 10 "$CTX_CLR" "$L2_DIM")

# Effort level color
case $EFFORT in
    max)    EFFORT_CLR="\033[38;2;150;210;150m" ;;  # sage
    high)   EFFORT_CLR="\033[38;2;150;210;150m" ;;  # sage
    medium) EFFORT_CLR="\033[38;2;170;170;170m" ;;  # gray (blends in)
    low)    EFFORT_CLR="\033[38;2;225;150;150m" ;;  # coral (warning)
esac

L2C="${RST}\033[38;2;0;0;0m${NF_CORNER_BL}${BG2} ${L2_TXT}${NF_MODEL} ${MODEL} ${EFFORT_CLR}${EFFORT}${B2} ${L2_DIM}│${B2} ${L2_TXT}${NF_CLOCK} ${TIME_CLR}${TIME}${B2} ${L2_DIM}│${B2} ${CTX_BAR} ${CTX_CLR}${PCT}%${B2} ${L2_TXT}of ${CTX_SIZE_K}k"

FIVE_PCT=$(echo "$DATA" | jq -r '.rate_limits.five_hour.used_percentage // empty' 2>/dev/null) || true
SEVEN_PCT=$(echo "$DATA" | jq -r '.rate_limits.seven_day.used_percentage // empty' 2>/dev/null) || true
FIVE_RESET_TS=$(echo "$DATA" | jq -r '.rate_limits.five_hour.resets_at // empty' 2>/dev/null) || true
SEVEN_RESET_TS=$(echo "$DATA" | jq -r '.rate_limits.seven_day.resets_at // empty' 2>/dev/null) || true

if [ -n "${FIVE_PCT:-}" ] && [ -n "${SEVEN_PCT:-}" ]; then
    FIVE_CLR=$(pct_txt_color "$FIVE_PCT")
    FIVE_BAR=$(make_bar "$FIVE_PCT" 5 "$FIVE_CLR" "$L2_DIM")
    FIVE_TIME=$(format_reset "$FIVE_RESET_TS")
    L2C+=" ${L2_DIM}│${B2} ${L2_TXT}5h ${FIVE_BAR} ${FIVE_CLR}${FIVE_PCT}%${B2}"
    [ -n "${FIVE_TIME:-}" ] && L2C+=" ${L2_DIM}${FIVE_TIME}${B2}"

    SEVEN_CLR=$(pct_txt_color "$SEVEN_PCT")
    SEVEN_BAR=$(make_bar "$SEVEN_PCT" 5 "$SEVEN_CLR" "$L2_DIM")
    SEVEN_TIME=$(format_reset "$SEVEN_RESET_TS")
    L2C+=" ${L2_DIM}│${B2} ${L2_TXT}7d ${SEVEN_BAR} ${SEVEN_CLR}${SEVEN_PCT}%${B2}"
    [ -n "${SEVEN_TIME:-}" ] && L2C+=" ${L2_DIM}${SEVEN_TIME}${B2}"
fi

# ── Claude service status (auto-refresh every 60s in background) ────────────
SVC_CACHE="/tmp/claude-service-status"
SVC_FETCH="$HOME/.claude/claude-status-fetch.sh"
if [ -x "$SVC_FETCH" ]; then
    SVC_AGE=9999
    [ -f "$SVC_CACHE" ] && SVC_AGE=$(($(date +%s) - $(stat -f %m "$SVC_CACHE" 2>/dev/null || echo 0)))
    if [ "$SVC_AGE" -ge 60 ]; then
        ("$SVC_FETCH" &) 2>/dev/null
    fi
fi
if [ -f "$SVC_CACHE" ]; then
    SVC_RAW=$(head -1 "$SVC_CACHE" 2>/dev/null)
    case "$SVC_RAW" in
        operational)
            SVC_CLR="\033[38;2;100;200;120m"   # sage green
            SVC_ICON="✓"
            SVC_LABEL="ok"
            L2C+=" ${L2_DIM}│${B2} ${SVC_CLR}${SVC_ICON} ${SVC_LABEL}${B2}"
            ;;
        incident:*)
            INCIDENT_TITLE="${SVC_RAW#incident:}"
            SVC_CLR="\033[38;2;225;150;100m"   # coral/amber
            SVC_ICON="⚠"
            L2C+=" ${L2_DIM}│${B2} ${SVC_CLR}${SVC_ICON} ${INCIDENT_TITLE}${B2}"
            ;;
        degraded_performance:*|partial_outage:*|major_outage:*)
            SVC_IND="${SVC_RAW%%:*}"
            SVC_DESC="${SVC_RAW#*:}"
            SVC_DESC="${SVC_DESC%%:*}"
            case "$SVC_IND" in
                major_outage)         SVC_CLR="\033[38;2;225;100;100m" ; SVC_ICON="✗" ;;
                partial_outage)       SVC_CLR="\033[38;2;225;150;100m" ; SVC_ICON="⚠" ;;
                degraded_performance) SVC_CLR="\033[38;2;215;195;125m" ; SVC_ICON="~" ;;
                *)                    SVC_CLR="\033[38;2;170;170;170m" ; SVC_ICON="?" ;;
            esac
            SVC_LABEL=$(echo "$SVC_DESC" | cut -c1-40)
            L2C+=" ${L2_DIM}│${B2} ${SVC_CLR}${SVC_ICON} ${SVC_LABEL}${B2}"
            ;;
    esac
fi

L2C+=" "

# ── Match line lengths: pad shorter line ──────────────────────────────────
L1_COLS=$(count_cols "$L1C")
L2_COLS=$(count_cols "$L2C")
if [ "${L1_COLS:-0}" -gt 0 ] && [ "${L2_COLS:-0}" -gt 0 ]; then
    DIFF=$((L2_COLS - L1_COLS))
    if [ "$DIFF" -gt 0 ] && [ "$DIFF" -le 120 ]; then
        L1C+="${BG1}$(printf '%*s' "$DIFF" '')"
    elif [ "$DIFF" -lt 0 ] && [ "$DIFF" -ge -120 ]; then
        L2C+="${BG2}$(printf '%*s' "$((-DIFF))" '')"
    fi
fi

# ── Set terminal tab title ───────────────────────────────────────────────────
_TAB_TITLE="${TOPIC:-${DIR:-Claude}}"
printf '\033]1;%s\007' "$_TAB_TITLE" > /dev/tty 2>/dev/null || true

# ── Fill to terminal edge + diagonal corner cuts ──────────────────────────────
L2_END_FG="\033[38;2;0;0;0m"
echo -e "${L1C}\033[K\033[?7l\033[${COLS}G${RST}${PROJ_FG}${NF_CORNER_TR}\033[?7h"
echo -e "${L2C}\033[K\033[?7l\033[${COLS}G${RST}${L2_END_FG}${NF_CORNER_BR}\033[?7h"
```

{{% /details %}}

{{% details title="claude-status-fetch.sh (service status fetcher)" closed="true" %}}

```bash
#!/usr/bin/env bash
# Fetches Claude service status from status.claude.com and writes a cache file.
# Called by the statusline in the background when the cache is >60s old.
# Output file: /tmp/claude-service-status
# Format: one line, either:
#   operational
#   degraded_performance:<indicator>:<affected_components>
#   partial_outage:<indicator>:<affected_components>
#   major_outage:<indicator>:<affected_components>
#   incident:<incident_title>
#   unknown

set -euo pipefail

CACHE_FILE="/tmp/claude-service-status"
TMP_FILE="${CACHE_FILE}.tmp"

data=$(curl -s --max-time 8 \
    -H "Accept: application/json" \
    -H "User-Agent: claude-code-statusline/1.0" \
    "https://status.claude.com/api/v2/summary.json" 2>/dev/null) || {
    # On network error, leave existing cache intact
    exit 0
}

[ -z "$data" ] && exit 0

# Validate it's actually JSON with expected fields
echo "$data" | jq -e '.status.indicator' >/dev/null 2>&1 || exit 0

indicator=$(echo "$data" | jq -r '.status.indicator // "unknown"')
description=$(echo "$data" | jq -r '.status.description // ""')

# Check for active incidents
incident_count=$(echo "$data" | jq -r '.incidents | length' 2>/dev/null || echo "0")

if [ "$indicator" = "none" ] && [ "$incident_count" -eq 0 ]; then
    echo "operational" > "$TMP_FILE"
else
    if [ "$incident_count" -gt 0 ]; then
        # Get the first (most recent) incident name
        incident_name=$(echo "$data" | jq -r '.incidents[0].name // "Incident"')
        # Truncate long names
        incident_name=$(echo "$incident_name" | cut -c1-50)
        echo "incident:${incident_name}" > "$TMP_FILE"
    else
        # Degraded or outage without a named incident
        # Collect affected component names
        affected=$(echo "$data" | jq -r '
            [.components[]
             | select(.status != "operational")
             | .name
            ] | join(", ")' 2>/dev/null || echo "")
        affected=$(echo "$affected" | cut -c1-60)
        echo "${indicator}:${description}:${affected}" > "$TMP_FILE"
    fi
fi

mv "$TMP_FILE" "$CACHE_FILE"
```

{{% /details %}}

{{% details title="session-topic-capture.sh (hook)" closed="true" %}}

```bash
#!/usr/bin/env bash
# Session topic capture hook for Claude Code statusline v2
# Fires on UserPromptSubmit - calls Claude Haiku to generate a "Project: Focus" label
# Writes to ~/.claude/session-topics/{session_id}.txt
set -euo pipefail

HOOK_DATA=$(cat)

SESSION_ID=$(echo "$HOOK_DATA" | jq -r '.session_id // empty' 2>/dev/null)
[ -z "$SESSION_ID" ] && exit 0

TOPIC_DIR="$HOME/.claude/session-topics"
TOPIC_FILE="${TOPIC_DIR}/${SESSION_ID}.txt"
LOCK_FILE="/tmp/session-topic-${SESSION_ID}.lock"

# Get transcript path from hook data
TRANSCRIPT_PATH=$(echo "$HOOK_DATA" | jq -r '.transcript_path // empty' 2>/dev/null)

# Rate limit: only regenerate every 10 prompts or if no topic exists yet
COUNTER_FILE="/tmp/session-topic-counter-${SESSION_ID}"
COUNT=0
[ -f "$COUNTER_FILE" ] && COUNT=$(cat "$COUNTER_FILE" 2>/dev/null || echo 0)
COUNT=$((COUNT + 1))
echo "$COUNT" > "$COUNTER_FILE"

# Generate on prompt 1, every 10 prompts, or if no topic file exists
if [ -f "$TOPIC_FILE" ] && [ "$COUNT" -ne 1 ] && [ $((COUNT % 10)) -ne 0 ]; then
    exit 0
fi

# Prevent concurrent generation
if [ -f "$LOCK_FILE" ]; then
    lock_age=$(($(date +%s) - $(stat -f %m "$LOCK_FILE" 2>/dev/null || echo 0)))
    [ "$lock_age" -lt 30 ] && exit 0
fi
touch "$LOCK_FILE"

# Run in background so we don't block the prompt
(
    # Get OAuth token
    token=""
    creds=$(security find-generic-password -s "Claude Code-credentials" -w 2>/dev/null) && \
        token=$(echo "$creds" | jq -r '.claudeAiOauth.accessToken // empty' 2>/dev/null)
    if [ -z "$token" ]; then
        token=$(jq -r '.claudeAiOauth.accessToken // empty' "$HOME/.claude/.credentials.json" 2>/dev/null)
    fi
    [ -z "${token:-}" ] && { rm -f "$LOCK_FILE"; exit 0; }

    # Read transcript JSONL for context
    EXCERPT=""
    if [ -n "$TRANSCRIPT_PATH" ] && [ -f "$TRANSCRIPT_PATH" ]; then
        EXCERPT=$(tail -40 "$TRANSCRIPT_PATH" 2>/dev/null | \
            jq -r 'select(.type == "human" or .type == "assistant") |
                   if .type == "human" then "User: " + (.message // .content // "[prompt]" | tostring | .[0:200])
                   else "Assistant: " + (.message // .content // "[response]" | tostring | .[0:200])
                   end' 2>/dev/null | tail -20 | head -c 3000)
    fi

    # Fallback: use the prompt from hook data if transcript didn't yield anything
    if [ -z "$EXCERPT" ]; then
        PROMPT_TEXT=$(echo "$HOOK_DATA" | jq -r '.prompt // empty' 2>/dev/null)
        [ -n "$PROMPT_TEXT" ] && EXCERPT="User: ${PROMPT_TEXT:0:1000}"
    fi

    [ -z "$EXCERPT" ] && { rm -f "$LOCK_FILE"; exit 0; }

    # Get CWD for project context
    CWD=$(echo "$HOOK_DATA" | jq -r '.cwd // empty' 2>/dev/null)
    PROJECT_DIR=$(basename "${CWD:-unknown}")

    # Call Haiku for topic generation
    RESPONSE=$(curl -s --max-time 8 \
        -H "Authorization: Bearer $token" \
        -H "anthropic-beta: oauth-2025-04-20" \
        -H "Content-Type: application/json" \
        -H "anthropic-version: 2023-06-01" \
        -H "User-Agent: claude-code/2.1.4" \
        https://api.anthropic.com/v1/messages \
        -d "$(jq -n --arg excerpt "$EXCERPT" --arg project "$PROJECT_DIR" '{
            model: "claude-haiku-4-5-20251001",
            max_tokens: 30,
            messages: [{
                role: "user",
                content: ("Summarize this coding session in exactly the format \"Project: Focus\" where Project is the project/repo name (use \"" + $project + "\" if unclear) and Focus is a 1-3 word description of what is being worked on. Reply with ONLY the label, nothing else.\n\nConversation:\n" + $excerpt)
            }]
        }')" 2>/dev/null)

    TOPIC=$(echo "$RESPONSE" | jq -r '.content[0].text // empty' 2>/dev/null | tr -d '\n' | cut -c1-40)

    # Reject topics that look like JSON (Haiku sometimes mimics JSON from transcript)
    if [[ "$TOPIC" =~ ^\{ ]] || [[ "$TOPIC" =~ ^\[ ]]; then
        rm -f "$LOCK_FILE"
        exit 0
    fi

    if [ -n "$TOPIC" ]; then
        mkdir -p "$TOPIC_DIR"
        echo "$TOPIC" > "$TOPIC_FILE"
    fi

    rm -f "$LOCK_FILE"
) &
disown 2>/dev/null

exit 0
```

{{% /details %}}

### Credits

The v2 design is based on [lee-fuhr's statusline gist](https://gist.github.com/lee-fuhr/68141b3ad716a96950cd111c749442b6), adapted with my own icon preferences and added Kubernetes context.

---

## v1: Powerline Style

The original version uses a classic powerline layout with fixed colors. Single line, each segment has its own background color with arrow separators.

![Claude Code powerline status line](/images/claude-statusline.png)

Left to right:

- **Model** (Opus 4.6) - active Claude model
- **Folder** (git-manager/dev) - current repo as parent/folder
- **Git branch** (dev) - only shows when not on `main`
- **Kubernetes context** (k8s-blue-cc) - active cluster
- **Context window** (86%) - conversation fullness, scaled so 100% = compaction threshold
- **5h usage** (2%, resets in 4h6m) - rolling 5-hour API quota
- **7d usage** (53%, resets Thu 11:00) - rolling 7-day API quota

Each meter is color-coded: green under 50%, yellow 50-79%, red 80%+.

{{% details title="statusline.sh (v1)" closed="true" %}}

```bash
#!/bin/bash

# Modern Powerline-style statusline for Claude Code
# Requires: Nerd Font (for icons and separators)

# Read JSON input from stdin
input=$(cat)

# ═══════════════════════════════════════════════════════════════════════════════
# POWERLINE CHARACTERS & COLORS
# ═══════════════════════════════════════════════════════════════════════════════

# Powerline separators (Nerd Font)
SEP=""      # U+E0B0 - right arrow (filled)
SEPR=""     # U+E0B2 - left arrow (filled)

# Color definitions (256-color mode)
# Format: \033[38;5;XXXm = foreground, \033[48;5;XXXm = background
RST='\033[0m'

# Segment colors (bg, fg pairs)
# Using a cohesive dark palette
BG_MODEL="\033[48;5;24m"     # Deep blue
FG_MODEL="\033[38;5;255m"    # White text
FG_MODEL_SEP="\033[38;5;24m" # For separator after

BG_FOLDER="\033[48;5;30m"    # Teal
FG_FOLDER="\033[38;5;255m"   # White
FG_FOLDER_SEP="\033[38;5;30m"

BG_BRANCH="\033[48;5;132m"    # Mauve (git branch segment)
FG_BRANCH="\033[38;5;255m"   # White
FG_BRANCH_SEP="\033[38;5;132m"

BG_K8S="\033[48;5;91m"       # Purple
FG_K8S="\033[38;5;255m"      # White
FG_K8S_SEP="\033[38;5;91m"

BG_CTX="\033[48;5;237m"      # Dark grey
FG_CTX="\033[38;5;250m"      # Light grey
FG_CTX_SEP="\033[38;5;237m"

BG_5H="\033[48;5;236m"       # Darker grey
FG_5H="\033[38;5;250m"
FG_5H_SEP="\033[38;5;236m"

BG_7D="\033[48;5;235m"       # Darkest grey
FG_7D="\033[38;5;250m"
FG_7D_SEP="\033[38;5;235m"

# Status colors for usage levels
FG_OK="\033[38;5;77m"        # Green
FG_WARN="\033[38;5;220m"     # Yellow
FG_CRIT="\033[38;5;203m"     # Red

# Nerd Font icons
ICON_MODEL="󰚩"    # nf-md-robot
ICON_FOLDER="󰉋"   # nf-md-folder
ICON_GIT="󰊢"      # nf-md-git
ICON_BRANCH=""   # nf-oct-git_branch
ICON_K8S="󱃾"      # nf-md-kubernetes
ICON_CTX=""     # nf-fa-tachometer_alt / gauge
ICON_5H=""      # nf-fa-clock
ICON_7D=""      # nf-fa-calendar_week

# ═══════════════════════════════════════════════════════════════════════════════
# DATA EXTRACTION
# ═══════════════════════════════════════════════════════════════════════════════

# Extract basic info
model=$(echo "$input" | jq -r '.model.display_name')
cwd=$(echo "$input" | jq -r '.workspace.current_dir')

# Get folder and branch (separate segments)
folder="$(basename "$(dirname "$cwd")")/$(basename "$cwd")"
folder_icon="${ICON_FOLDER}"
show_branch=""
branch_name=""

if git -C "$cwd" rev-parse --is-inside-work-tree &>/dev/null; then
    branch_name=$(git -C "$cwd" branch --show-current 2>/dev/null)
    if [[ "$branch_name" != "main" && -n "$branch_name" ]]; then
        show_branch="yes"
    fi
fi

# Context window usage
context_size=$(echo "$input" | jq -r '.context_window.context_window_size // 0')
context_usage=$(echo "$input" | jq -r '.context_window.current_usage // null')
if [[ "$context_usage" != "null" && "$context_size" -gt 0 ]]; then
    context_tokens=$(echo "$input" | jq -r '.context_window.current_usage | (.input_tokens // 0) + (.cache_creation_input_tokens // 0) + (.cache_read_input_tokens // 0)')
    context_pct_raw=$((context_tokens * 100 / context_size))
    # Scale so 82% (conservative threshold) displays as 100%
    # This gives more headroom before compaction at 78%
    context_pct=$((context_pct_raw * 100 / 82))
    [[ $context_pct -gt 100 ]] && context_pct=100
else
    context_pct=0
fi

# Kubernetes context
k8s_context=$(kubectl config current-context 2>/dev/null || echo "none")

# ═══════════════════════════════════════════════════════════════════════════════
# API USAGE (5h/7d from Anthropic API)
# ═══════════════════════════════════════════════════════════════════════════════

USAGE_CACHE="$HOME/.cache/cc-usage.txt"
USAGE_LOCK="$HOME/.cache/cc-usage.lock"
USAGE_TTL="${CC_CACHE_TTL:-60}"
mkdir -p "$HOME/.cache"

get_api_usage() {
    if [[ -f "$USAGE_CACHE" ]]; then
        age=$(($(date +%s) - $(stat -f '%m' "$USAGE_CACHE" 2>/dev/null || echo 0)))
        if [[ $age -lt $USAGE_TTL ]]; then
            cat "$USAGE_CACHE"
            return
        fi
    fi

    if [[ -f "$USAGE_LOCK" ]]; then
        age=$(($(date +%s) - $(stat -f '%m' "$USAGE_LOCK" 2>/dev/null || echo 0)))
        if [[ $age -lt 30 ]]; then
            [[ -f "$USAGE_CACHE" ]] && cat "$USAGE_CACHE"
            return
        fi
    fi

    touch "$USAGE_LOCK"

    creds=$(security find-generic-password -s "Claude Code-credentials" -w 2>/dev/null)
    if [[ -z "$creds" ]]; then
        echo "?|?|?|?"
        return
    fi

    token=$(echo "$creds" | jq -r '.claudeAiOauth.accessToken // empty' 2>/dev/null)
    if [[ -z "$token" ]]; then
        echo "?|?|?|?"
        return
    fi

    resp=$(curl -s --max-time 5 \
        "https://api.anthropic.com/api/oauth/usage" \
        -H "Authorization: Bearer $token" \
        -H "anthropic-beta: oauth-2025-04-20" 2>/dev/null) || true

    if [[ -z "$resp" ]]; then
        [[ -f "$USAGE_CACHE" ]] && cat "$USAGE_CACHE" || echo "?|?|?|?"
        return
    fi

    session=$(echo "$resp" | jq -r '.five_hour.utilization // empty' 2>/dev/null)
    weekly=$(echo "$resp" | jq -r '.seven_day.utilization // empty' 2>/dev/null)

    if [[ -z "$session" || -z "$weekly" ]]; then
        [[ -f "$USAGE_CACHE" ]] && cat "$USAGE_CACHE" || echo "?|?|?|?"
        return
    fi

    session_int=$(printf "%.0f" "$session")
    weekly_int=$(printf "%.0f" "$weekly")

    session_reset=$(echo "$resp" | jq -r '.five_hour.resets_at // empty' 2>/dev/null)
    if [[ -n "$session_reset" ]]; then
        utc_epoch_5h=$(date -j -u -f "%Y-%m-%dT%H:%M:%S" "${session_reset%%.*}" "+%s" 2>/dev/null)
        if [[ -n "$utc_epoch_5h" ]]; then
            now_epoch=$(date +%s)
            diff_sec=$((utc_epoch_5h - now_epoch))
            if [[ $diff_sec -gt 0 ]]; then
                diff_hr=$((diff_sec / 3600))
                diff_min=$(((diff_sec % 3600) / 60))
                session_reset_fmt="${diff_hr}h${diff_min}m"
            else
                session_reset_fmt="now"
            fi
        else
            session_reset_fmt="?"
        fi
    else
        session_reset_fmt="?"
    fi

    weekly_reset=$(echo "$resp" | jq -r '.seven_day.resets_at // empty' 2>/dev/null)
    if [[ -n "$weekly_reset" ]]; then
        utc_epoch=$(date -j -u -f "%Y-%m-%dT%H:%M:%S" "${weekly_reset%%.*}" "+%s" 2>/dev/null)
        if [[ -n "$utc_epoch" ]]; then
            weekly_reset_fmt=$(date -j -f "%s" "$utc_epoch" "+%a %H:%M" 2>/dev/null || echo "?")
        else
            weekly_reset_fmt="?"
        fi
    else
        weekly_reset_fmt="?"
    fi

    echo "${session_int}|${weekly_int}|${session_reset_fmt}|${weekly_reset_fmt}" | tee "$USAGE_CACHE"
}

api_usage_raw=$(get_api_usage)
usage_5h=$(echo "$api_usage_raw" | cut -d'|' -f1)
usage_7d=$(echo "$api_usage_raw" | cut -d'|' -f2)
reset_5h=$(echo "$api_usage_raw" | cut -d'|' -f3)
reset_7d=$(echo "$api_usage_raw" | cut -d'|' -f4)

# ═══════════════════════════════════════════════════════════════════════════════
# PROGRESS BAR (compact dot style)
# ═══════════════════════════════════════════════════════════════════════════════

progress_bar() {
    local percent=$1
    local width=5

    if ! [[ "$percent" =~ ^[0-9]+$ ]]; then
        printf "○○○○○"
        return
    fi

    local filled=$((percent * width / 100))
    [[ $filled -gt $width ]] && filled=$width
    [[ $filled -lt 0 ]] && filled=0
    local empty=$((width - filled))

    local bar=""
    for ((i=0; i<filled; i++)); do bar+="●"; done
    for ((i=0; i<empty; i++)); do bar+="○"; done
    printf "%s" "$bar"
}

# Get color based on percentage
get_usage_color() {
    local pct=$1
    if ! [[ "$pct" =~ ^[0-9]+$ ]]; then
        echo "$FG_CTX"
        return
    fi
    if [[ $pct -ge 80 ]]; then
        echo "$FG_CRIT"
    elif [[ $pct -ge 50 ]]; then
        echo "$FG_WARN"
    else
        echo "$FG_OK"
    fi
}

# Generate bars with colors
bar_ctx=$(progress_bar "$context_pct")
bar_5h=$(progress_bar "$usage_5h")
bar_7d=$(progress_bar "$usage_7d")

color_ctx=$(get_usage_color "$context_pct")
color_5h=$(get_usage_color "$usage_5h")
color_7d=$(get_usage_color "$usage_7d")

# ═══════════════════════════════════════════════════════════════════════════════
# BUILD POWERLINE OUTPUT
# ═══════════════════════════════════════════════════════════════════════════════

# Segment 1: Model
printf "${BG_MODEL}${FG_MODEL} ${ICON_MODEL} %s " "$model"
printf "${BG_FOLDER}${FG_MODEL_SEP}${SEP}"

# Segment 2: Folder
printf "${BG_FOLDER}${FG_FOLDER} ${folder_icon} %s " "$folder"

# Segment 3: Branch (only if not on main)
if [[ -n "$show_branch" ]]; then
    printf "${BG_BRANCH}${FG_FOLDER_SEP}${SEP}"
    printf "${BG_BRANCH}${FG_BRANCH} ${ICON_GIT}${ICON_BRANCH} %s " "$branch_name"
    printf "${BG_K8S}${FG_BRANCH_SEP}${SEP}"
else
    printf "${BG_K8S}${FG_FOLDER_SEP}${SEP}"
fi

# Segment 4: Kubernetes
printf "${BG_K8S}${FG_K8S} ${ICON_K8S} %s " "$k8s_context"
printf "${BG_CTX}${FG_K8S_SEP}${SEP}"

# Segment 4: Context usage
printf "${BG_CTX}${FG_CTX} ${ICON_CTX} ${color_ctx}%s${FG_CTX} %s%% " "$bar_ctx" "$context_pct"
printf "${BG_5H}${FG_CTX_SEP}${SEP}"

# Segment 5: 5h usage
printf "${BG_5H}${FG_5H} ${ICON_5H} ${color_5h}%s${FG_5H} %s%% %s " "$bar_5h" "$usage_5h" "$reset_5h"
printf "${BG_7D}${FG_5H_SEP}${SEP}"

# Segment 6: 7d usage
printf "${BG_7D}${FG_7D} ${ICON_7D} ${color_7d}%s${FG_7D} %s%% %s " "$bar_7d" "$usage_7d" "$reset_7d"
printf "${RST}${FG_7D_SEP}${SEP}${RST}"
```

{{% /details %}}

## How It Works

Claude Code calls your script after each assistant message, piping a JSON blob to stdin with session metadata (`model`, `cwd`, `context_window` with token breakdown, `cost`, `session_id`, etc.). Your script reads it and prints ANSI-colored output to stdout.

{{< callout type="info" >}}
The API usage part reads the OAuth token from macOS Keychain via `security find-generic-password`. On Linux, you'd need a different approach to read the token.
{{< /callout >}}

## Adapting It

Each segment is independent. Ask Claude Code to adapt the script to your needs, or use it as a starting point for your own design.

