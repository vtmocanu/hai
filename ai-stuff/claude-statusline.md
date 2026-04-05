# Custom Claude Code Status Line

Claude Code has a fully customizable status line. You point it at a shell script, it pipes in session data as JSON, and your script renders whatever you want. The current version is **v2.1.0** (see the v1 Powerline style in the tabs below).

The v2 layout uses two lines with diagonal corner cuts and width-synchronized lines. Each session gets a unique color from a 12-color palette (hashed from the session ID), so when I have multiple sessions open, I can tell them apart at a glance.

![Claude Code statusline v2](/images/claude-statusline-v2.png)

**Line 1** (project-colored background): session topic, folder, git branch + status, Kubernetes context

**Line 2** (black background): model + effort level, elapsed time, context window bar, 5h/7d API quota bars with reset times, Claude service status icon

All meters are color-coded: green under 50%, gold 50-80%, coral 80%+.

{{< tabs >}}
{{< tab name="Installation" icon="lightning-bolt" >}}

## Quick Start

{{< callout type="info" >}}
The fastest way to get this running: open Claude Code and paste this page's URL with "implement this statusline for me". It will read the scripts, save them, configure `settings.json`, and set up the hook.
{{< /callout >}}

**Requirements:** [Nerd Font](https://www.nerdfonts.com/) v3+ (for powerline corners and icons), `jq`, `kubectl` (optional). Claude Code v2.1.80+ for native rate limit data.

### 1. Save the scripts

Save the three scripts from the "Scripts" section below and make them executable:

```bash
chmod +x ~/.claude/statusline.sh
chmod +x ~/.claude/claude-status-fetch.sh
chmod +x ~/.claude/hooks/session-topic-capture.sh
mkdir -p ~/.claude/session-topics
```

### 2. Configure Claude Code

Add to `~/.claude/settings.json`:

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

### 3. Restart Claude Code

The topic will appear after your first prompt (it runs async, so it shows on the second render).

### Scripts

{{% details title="statusline.sh (v2.1.0)" closed="true" %}}

```bash
#!/usr/bin/env bash
set -uo pipefail  # no -e: external commands (git, kubectl, jq) can fail; silent crash = no statusline
trap 'printf "\n"' EXIT  # ensure at least empty output on crash
[ "${STATUSLINE_DEBUG:-}" = "1" ] && exec 2>/tmp/statusline-debug.log

DATA=$(timeout 2 cat 2>/dev/null) || DATA=""
[ -z "$DATA" ] && exit 0

# ── Extract ALL fields in a single jq call ──────────────────────────────────
# Uses jq @sh to produce shell-safe quoted assignments. No IFS tricks needed;
# empty fields become VAR='' instead of being silently swallowed.
eval "$(echo "$DATA" | jq -r '
    @sh "MODEL=\(.model.display_name // "Claude" | gsub(" \\(.*\\)"; ""))",
    @sh "MODEL_ID=\(try (.model.id // "unknown") catch "unknown")",
    @sh "DIR=\(.cwd // "~" | split("/") | .[-2:] | join("/"))",
    @sh "PCT=\(try (
        if (.context_window.remaining_percentage // null) != null then
            100 - (.context_window.remaining_percentage | floor)
        elif (.context_window.context_window_size // 0) > 0 then
            (((.context_window.current_usage.input_tokens // 0) +
              (.context_window.current_usage.cache_creation_input_tokens // 0) +
              (.context_window.current_usage.cache_read_input_tokens // 0)) * 100 /
             .context_window.context_window_size) | floor
        else 0 end
    ) catch 0)",
    @sh "CTX_SIZE=\(.context_window.context_window_size // 200000)",
    @sh "DURATION_MS=\(.cost.total_duration_ms // 0)",
    @sh "AGENT=\(.agent.name // "")",
    @sh "MODE=\(.mode // "")",
    @sh "TRANSCRIPT_PATH=\(.transcript_path // "")",
    @sh "CWD_FULL=\(.cwd // "~")",
    @sh "SESSION_ID=\(.session_id // "")",
    @sh "FIVE_PCT=\(.rate_limits.five_hour.used_percentage // "")",
    @sh "SEVEN_PCT=\(.rate_limits.seven_day.used_percentage // "")",
    @sh "FIVE_RESET_TS=\(.rate_limits.five_hour.resets_at // "")",
    @sh "SEVEN_RESET_TS=\(.rate_limits.seven_day.resets_at // "")"
' 2>/dev/null)" 2>/dev/null

# Guard: if jq failed completely, use safe defaults
MODEL=${MODEL:-Claude}; MODEL_ID=${MODEL_ID:-unknown}; DIR=${DIR:-~}
PCT=${PCT:-0}; CTX_SIZE=${CTX_SIZE:-200000}; DURATION_MS=${DURATION_MS:-0}
AGENT=${AGENT:-}; MODE=${MODE:-}; TRANSCRIPT_PATH=${TRANSCRIPT_PATH:-}
CWD_FULL=${CWD_FULL:-~}; SESSION_ID=${SESSION_ID:-}
FIVE_PCT=${FIVE_PCT:-}; SEVEN_PCT=${SEVEN_PCT:-}
FIVE_RESET_TS=${FIVE_RESET_TS:-}; SEVEN_RESET_TS=${SEVEN_RESET_TS:-}

PCT=${PCT%%.*}  # truncate jq float rounding (e.g. 14.000000000000002 -> 14)
FIVE_PCT=${FIVE_PCT%%.*}   # also truncate rate limit floats
SEVEN_PCT=${SEVEN_PCT%%.*}
CTX_SIZE_K=$((CTX_SIZE / 1000))
# Max line width before Claude Code's cli-truncate drops line 2
SAFE_WIDTH=${STATUSLINE_WIDTH:-110}

TOPIC=""  # populated after SESSION_ID is extracted below

# ── Effort level detection (transcript -> settings -> default) ──────────────
EFFORT=""
if [ -n "$TRANSCRIPT_PATH" ] && [ -f "$TRANSCRIPT_PATH" ]; then
    # Read from end of file for speed on large transcripts
    EFFORT=$(tail -r "$TRANSCRIPT_PATH" 2>/dev/null \
        | grep -m1 -E '"content":"<local-command-stdout>(Set model to.*effort|Set effort level to)' \
        | grep -oE '\b(low|medium|high|max)\b' | tail -1 || true)
fi
if [ -z "$EFFORT" ]; then
    EFFORT=$(jq -r '.effortLevel // empty' "$HOME/.claude/settings.json" 2>/dev/null || true)
fi
EFFORT=${EFFORT:-medium}

# ── Nerd Font icons ───────────────────────────────────────────────────────
NF_GIT=$'\xee\x82\xa0'       # U+E0A0 powerline branch
NF_FOLDER="󰉋"               # nf-md-folder (kept from v1)
NF_MODEL="󰚩"                # nf-md-robot (kept from v1)
NF_K8S="󱃾"                  # nf-md-kubernetes (kept from v1)
NF_CLOCK=$'\xef\x80\x97'     # U+F017 clock
NF_CORNER_TL=$'\xee\x82\xba'    # U+E0BA lower-right fill (top-left corner)
NF_CORNER_BL=$'\xee\x82\xbe'    # U+E0BE upper-right fill (bottom-left corner)
NF_CORNER_TR=$'\xee\x82\xb8'    # U+E0B8 lower-left fill -> top-right corner cut
NF_CORNER_BR=$'\xee\x82\xbc'    # U+E0BC upper-left fill -> bottom-right corner cut

# ── Project-colored background (hash session ID -> unique hue) ────────────
RST="\033[0m"
PROJECT_ROOT=$(git -C "$CWD_FULL" rev-parse --show-toplevel 2>/dev/null || echo "$CWD_FULL")
PHASH=$(printf '%s' "${SESSION_ID:-$CWD_FULL}" | cksum | cut -d' ' -f1 || echo "0")

# ── Session topic ─────────────────────────────────────────────────────────
if [ -n "${SESSION_ID:-}" ]; then
    TOPIC_FILE="$HOME/.claude/session-topics/${SESSION_ID}.txt"
    if [ -f "$TOPIC_FILE" ]; then
        # Strip ANSI escape sequences and limit to 40 chars
        TOPIC=$(cat "$TOPIC_FILE" 2>/dev/null | tr -d '\n' | gsed 's/\x1b\[[0-9;]*m//g' 2>/dev/null | cut -c1-40)
    fi
fi

# Check for manual color override
COLOR_OVERRIDES="$HOME/.claude/statusline-color-overrides.json"
if [ -f "$COLOR_OVERRIDES" ]; then
    COLOR_IDX=$(jq -r --arg p "$PROJECT_ROOT" '.[$p] // empty' "$COLOR_OVERRIDES" 2>/dev/null || true)
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
    local p=${1:-0}
    p=${p%%.*}  # safety: strip decimal if any
    if   [ "${p:-0}" -gt 80 ] 2>/dev/null; then printf "\033[38;2;225;150;150m"   # coral
    elif [ "${p:-0}" -gt 50 ] 2>/dev/null; then printf "\033[38;2;215;195;125m"   # gold
    else                                         printf "\033[38;2;150;210;150m"   # sage
    fi
}

# ── Git info ────────────────────────────────────────────────────────────────
BRANCH=$(git -c core.useBuiltinFSMonitor=false branch --show-current 2>/dev/null || echo "")
GIT_STATUS=""
if [ -n "$BRANCH" ]; then
  STAGED=$(git diff --cached --numstat 2>/dev/null | wc -l | tr -d " ")
  MODIFIED=$(git diff --numstat 2>/dev/null | wc -l | tr -d " ")
  UNTRACKED=$(git ls-files --others --exclude-standard 2>/dev/null | wc -l | tr -d " ")
  [ "${STAGED:-0}" -gt 0 ]    && GIT_STATUS="+${STAGED}"
  [ "${MODIFIED:-0}" -gt 0 ]  && GIT_STATUS="${GIT_STATUS:+$GIT_STATUS }!${MODIFIED}"
  [ "${UNTRACKED:-0}" -gt 0 ] && GIT_STATUS="${GIT_STATUS:+$GIT_STATUS }?${UNTRACKED}"
fi

# ── Kubernetes context (with timeout to avoid exec-auth hangs) ──────────────
K8S_CTX=$(timeout 2 kubectl config current-context 2>/dev/null || echo "")

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
    local pct=${1:-0} width=${2:-5} fill_clr="$3" empty_clr="$4" bar=""
    pct=${pct%%.*}  # safety: strip decimal
    [ -z "$pct" ] && pct=0
    local filled=$((pct * width / 100))
    [ "$pct" -gt 0 ] 2>/dev/null && [ "$filled" -eq 0 ] && filled=1
    [ "$filled" -gt "$width" ] && filled=$width
    [ "$filled" -lt 0 ] && filled=0
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

# ── Count visible columns (ANSI-aware, multi-string in one perl call) ──────
# Usage: measure_cols "str1" "str2" ... -> outputs one number per line
measure_cols() {
    local args=()
    for s in "$@"; do args+=("$(printf '%b' "$s")"); done
    printf '%s\n' "${args[@]}" | perl -ne '
        s/\e\[[0-9;]*m//g;
        chomp;
        use Encode qw(decode);
        my $decoded = decode("UTF-8", $_, Encode::FB_DEFAULT);
        print length($decoded), "\n";
    ' 2>/dev/null
}

# ── Line 1: build with bash-based width tracking (no perl) ─────────────────
# Estimate visible width per component from known text lengths.
# Each component's ANSI overhead cancels out; we just count visible chars.
# Corner char = 1, each separator "│" = 1, icon = 1-2, spaces counted explicitly.
# Add 3-char safety margin for Nerd Font icons that render wider than 1 codepoint.
L1_EST=5  # corner char(1) + trailing space(1) + icon safety margin(3)

[ -n "$TOPIC" ] && L1_EST=$((L1_EST + 1 + ${#TOPIC} + 3))   # " TOPIC │"
L1_EST=$((L1_EST + 1 + 2 + ${#DIR} + 1))                     # " icon DIR "
if [ -n "$BRANCH" ]; then
    L1_EST=$((L1_EST + 1 + 1 + 2 + ${#BRANCH}))              # "│ icon BRANCH"
    [ -n "$GIT_STATUS" ] && L1_EST=$((L1_EST + 1 + ${#GIT_STATUS}))
fi
[ -n "$AGENT" ] && L1_EST=$((L1_EST + 1 + ${#AGENT}))
[ -n "$MODE" ]  && L1_EST=$((L1_EST + 3 + ${#MODE}))         # " │ MODE"
[ -n "$K8S_CTX" ] && L1_EST=$((L1_EST + 1 + 1 + 2 + ${#K8S_CTX}))  # " │ icon K8S"

# Truncate if over SAFE_WIDTH (order: K8S > BRANCH > TOPIC)
if [ "$L1_EST" -gt "$SAFE_WIDTH" ] && [ -n "$K8S_CTX" ]; then
    OVER=$((L1_EST - SAFE_WIDTH))
    K8S_MAX=$((${#K8S_CTX} - OVER - 2))
    if [ "$K8S_MAX" -gt 5 ]; then
        K8S_CTX="${K8S_CTX:0:$K8S_MAX}.."
    else
        K8S_CTX=""
    fi
    # Recalculate
    L1_EST=2; [ -n "$TOPIC" ] && L1_EST=$((L1_EST + 1 + ${#TOPIC} + 3))
    L1_EST=$((L1_EST + 1 + 2 + ${#DIR} + 1))
    [ -n "$BRANCH" ] && { L1_EST=$((L1_EST + 1 + 1 + 2 + ${#BRANCH})); [ -n "$GIT_STATUS" ] && L1_EST=$((L1_EST + 1 + ${#GIT_STATUS})); }
    [ -n "$AGENT" ] && L1_EST=$((L1_EST + 1 + ${#AGENT}))
    [ -n "$MODE" ]  && L1_EST=$((L1_EST + 3 + ${#MODE}))
    [ -n "$K8S_CTX" ] && L1_EST=$((L1_EST + 1 + 1 + 2 + ${#K8S_CTX}))
fi

if [ "$L1_EST" -gt "$SAFE_WIDTH" ] && [ -n "$BRANCH" ]; then
    OVER=$((L1_EST - SAFE_WIDTH))
    BR_MAX=$((${#BRANCH} - OVER - 2))
    if [ "$BR_MAX" -gt 5 ]; then
        BRANCH="${BRANCH:0:$BR_MAX}.."
    else
        BRANCH="${BRANCH:0:8}.."
    fi
    # No need to recalculate again; next truncation target (TOPIC) is rare
fi

if [ "$L1_EST" -gt "$SAFE_WIDTH" ] && [ -n "$TOPIC" ]; then
    # Recalculate after branch truncation
    L1_EST=2; [ -n "$TOPIC" ] && L1_EST=$((L1_EST + 1 + ${#TOPIC} + 3))
    L1_EST=$((L1_EST + 1 + 2 + ${#DIR} + 1))
    [ -n "$BRANCH" ] && { L1_EST=$((L1_EST + 1 + 1 + 2 + ${#BRANCH})); [ -n "$GIT_STATUS" ] && L1_EST=$((L1_EST + 1 + ${#GIT_STATUS})); }
    [ -n "$AGENT" ] && L1_EST=$((L1_EST + 1 + ${#AGENT}))
    [ -n "$MODE" ]  && L1_EST=$((L1_EST + 3 + ${#MODE}))
    [ -n "$K8S_CTX" ] && L1_EST=$((L1_EST + 1 + 1 + 2 + ${#K8S_CTX}))
    if [ "$L1_EST" -gt "$SAFE_WIDTH" ]; then
        OVER=$((L1_EST - SAFE_WIDTH))
        T_MAX=$((${#TOPIC} - OVER - 2))
        if [ "$T_MAX" -gt 5 ]; then
            TOPIC="${TOPIC:0:$T_MAX}.."
        else
            TOPIC=""
        fi
    fi
fi

# Assemble Line 1 from (possibly truncated) components
L1_PREFIX="${RST}${PROJ_FG}${NF_CORNER_TL}${BG1}"
L1C="${L1_PREFIX}"
[ -n "$TOPIC" ] && L1C+=" ${TXT_BOLD}${TOPIC}${B} ${SEP}${B}"
L1C+=" ${TXT_FG}${NF_FOLDER} ${DIR} ${B}"
if [ -n "$BRANCH" ]; then
    L1C+="${SEP}${B} ${TXT_FG}${NF_GIT} ${BRANCH}${B}"
    [ -n "$GIT_STATUS" ] && L1C+=" ${TXT_FG}${GIT_STATUS}${B}"
fi
[ -n "$AGENT" ] && L1C+=" ${TXT_FG}${AGENT}${B}"
[ -n "$MODE" ]  && L1C+=" ${SEP}${B} \033[1;38;2;150;100;0m${MODE}${B}"
[ -n "$K8S_CTX" ] && L1C+=" ${SEP}${B} ${TXT_FG}${NF_K8S} ${K8S_CTX}${B}"
L1C+=" "

# ── Line 2 content ──────────────────────────────────────────────────────────
CTX_CLR=$(pct_txt_color "$PCT")
CTX_BAR=$(make_bar "$PCT" 7 "$CTX_CLR" "$L2_DIM")
# Effort level color
case $EFFORT in
    max)    EFFORT_CLR="\033[38;2;150;210;150m" ;;  # sage
    high)   EFFORT_CLR="\033[38;2;150;210;150m" ;;  # sage
    medium) EFFORT_CLR="\033[38;2;170;170;170m" ;;  # gray (blends in)
    low)    EFFORT_CLR="\033[38;2;225;150;150m" ;;  # coral (warning)
    *)      EFFORT_CLR="\033[38;2;170;170;170m" ;;  # fallback: gray
esac

L2C="${RST}\033[38;2;0;0;0m${NF_CORNER_BL}${BG2} ${L2_TXT}${NF_MODEL} ${MODEL} ${L2_DIM}·${B2} ${EFFORT_CLR}${EFFORT}${B2} ${L2_DIM}│${B2} ${L2_TXT}${NF_CLOCK} ${TIME_CLR}${TIME}${B2} ${L2_DIM}│${B2} ${CTX_BAR} ${CTX_CLR}${PCT}%${B2} ${L2_TXT}of ${CTX_SIZE_K}k"

# Estimate L2 base width with bash (same approach as L1: count visible chars)
# Model + effort + separator + clock + time + separator + bar(7) + pct + "of Xk"
L2_BASE_W=$((2 + 2 + ${#MODEL} + 1 + ${#EFFORT} + 3 + 2 + ${#TIME} + 3 + 7 + 1 + ${#PCT} + 1 + 4 + ${#CTX_SIZE_K} + 1))
L2_BASE_W=${L2_BASE_W:-50}
RATE_AVAIL=$((SAFE_WIDTH - L2_BASE_W - 5))   # reserve 5 for incident icon

if [ -n "${FIVE_PCT:-}" ] && [ -n "${SEVEN_PCT:-}" ]; then
    FIVE_CLR=$(pct_txt_color "$FIVE_PCT")
    SEVEN_CLR=$(pct_txt_color "$SEVEN_PCT")

    if [ "$RATE_AVAIL" -gt 40 ] 2>/dev/null; then
        # Full: bars + pct + reset times
        FIVE_BAR=$(make_bar "$FIVE_PCT" 5 "$FIVE_CLR" "$L2_DIM")
        FIVE_TIME=$(format_reset "$FIVE_RESET_TS")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}5h ${FIVE_BAR} ${FIVE_CLR}${FIVE_PCT}%${B2}"
        [ -n "${FIVE_TIME:-}" ] && L2C+=" ${L2_DIM}${FIVE_TIME}${B2}"
        SEVEN_BAR=$(make_bar "$SEVEN_PCT" 5 "$SEVEN_CLR" "$L2_DIM")
        SEVEN_TIME=$(format_reset "$SEVEN_RESET_TS")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}7d ${SEVEN_BAR} ${SEVEN_CLR}${SEVEN_PCT}%${B2}"
        [ -n "${SEVEN_TIME:-}" ] && L2C+=" ${L2_DIM}${SEVEN_TIME}${B2}"
    elif [ "$RATE_AVAIL" -gt 25 ] 2>/dev/null; then
        # Compact: bars + pct, no reset times
        FIVE_BAR=$(make_bar "$FIVE_PCT" 5 "$FIVE_CLR" "$L2_DIM")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}5h ${FIVE_BAR} ${FIVE_CLR}${FIVE_PCT}%${B2}"
        SEVEN_BAR=$(make_bar "$SEVEN_PCT" 5 "$SEVEN_CLR" "$L2_DIM")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}7d ${SEVEN_BAR} ${SEVEN_CLR}${SEVEN_PCT}%${B2}"
    elif [ "$RATE_AVAIL" -gt 15 ] 2>/dev/null; then
        # Minimal: just percentages
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}5h ${FIVE_CLR}${FIVE_PCT}%${B2} ${L2_TXT}7d ${SEVEN_CLR}${SEVEN_PCT}%${B2}"
    fi
fi

# ── Claude service status (auto-refresh every 60s in background) ────────────
SVC_CACHE="/tmp/claude-service-status"
SVC_FETCH="$HOME/.claude/claude-status-fetch.sh"
if [ -x "$SVC_FETCH" ]; then
    SVC_AGE=9999
    [ -f "$SVC_CACHE" ] && SVC_AGE=$(($(date +%s) - $(stat -f %m "$SVC_CACHE" 2>/dev/null || echo 0)))
    if [ "$SVC_AGE" -ge 60 ]; then
        ("$SVC_FETCH" >/dev/null 2>/dev/null &)
    fi
fi
if [ -f "$SVC_CACHE" ]; then
    SVC_RAW=$(head -1 "$SVC_CACHE" 2>/dev/null)
    case "${SVC_RAW:-}" in
        operational)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;100;200;120m✓${B2}"
            ;;
        incident:*)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;225;150;100m⚠${B2}"
            ;;
        degraded_performance:*)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;215;195;125m~${B2}"
            ;;
        partial_outage:*|major_outage:*)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;225;100;100m✗${B2}"
            ;;
    esac
fi

L2C+=" "

# ── Set terminal tab title ───────────────────────────────────────────────────
_TAB_TITLE="${TOPIC:-${DIR:-Claude}}"
printf '\033]1;%s\007' "$_TAB_TITLE" > /dev/tty 2>/dev/null || true

# ── Pad shorter line to match longer ────────────────────────────────────────
{
    # Single perl invocation for both line measurements
    read -r L1_COLS L2_COLS < <(
        measure_cols "$L1C" "$L2C" | tr '\n' ' '
    )
    L1_COLS=${L1_COLS:-0}; L2_COLS=${L2_COLS:-0}
    SYNC_W=$L2_COLS
    [ "$L1_COLS" -gt "$SYNC_W" ] 2>/dev/null && SYNC_W=$L1_COLS
    if [ "$L1_COLS" -gt 10 ] 2>/dev/null && [ "$L1_COLS" -lt "$SYNC_W" ] 2>/dev/null; then
        L1C+="${BG1}$(printf '%*s' "$((SYNC_W - L1_COLS))" '')"
    fi
    if [ "$L2_COLS" -gt 10 ] 2>/dev/null && [ "$L2_COLS" -lt "$SYNC_W" ] 2>/dev/null; then
        L2C+="${BG2}$(printf '%*s' "$((SYNC_W - L2_COLS))" '')"
    fi
} 2>/dev/null || true

# ── Output ───────────────────────────────────────────────────────────────────
L2_END_FG="\033[38;2;0;0;0m"
trap - EXIT  # disarm crash trap before normal output
printf '\033[0m%b\n' "${L1C}${RST}${PROJ_FG}${NF_CORNER_TR}${RST}"
printf '\033[0m%b\n' "${L2C}${RST}${L2_END_FG}${NF_CORNER_BR}${RST}"
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

set -uo pipefail  # no -e: jq failures shouldn't leave orphan tmp files

CACHE_FILE="/tmp/claude-service-status"
TMP_FILE="${CACHE_FILE}.tmp"

# Clean up tmp file on any exit (crash, signal, normal)
trap 'rm -f "$TMP_FILE"' EXIT

data=$(curl -s --max-time 8 \
    -H "Accept: application/json" \
    -H "User-Agent: claude-code-statusline/1.0" \
    "https://status.claude.com/api/v2/summary.json" 2>/dev/null) || {
    # On network error, leave existing cache intact
    exit 0
}

[ -z "$data" ] && exit 0

# Extract all fields in a single jq call
eval "$(echo "$data" | jq -r '
    @sh "indicator=\(.status.indicator // "unknown")",
    @sh "description=\(.status.description // "")",
    @sh "incident_count=\(.incidents | length)",
    @sh "incident_name=\(.incidents[0].name // "Incident")"
' 2>/dev/null)" || exit 0

if [ "${indicator:-unknown}" = "none" ] && [ "${incident_count:-0}" -eq 0 ] 2>/dev/null; then
    echo "operational" > "$TMP_FILE"
else
    if [ "${incident_count:-0}" -gt 0 ] 2>/dev/null; then
        # Truncate long names
        incident_name=$(echo "${incident_name:-Incident}" | cut -c1-50)
        echo "incident:${incident_name}" > "$TMP_FILE"
    else
        # Degraded or outage without a named incident
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
trap - EXIT  # disarm cleanup since mv succeeded
```

{{% /details %}}

{{% details title="session-topic-capture.sh (hook)" closed="true" %}}

```bash
#!/usr/bin/env bash
# Session topic capture hook for Claude Code statusline v2
# Fires on UserPromptSubmit - calls Claude Haiku to generate a "Project: Focus" label
# Writes to ~/.claude/session-topics/{session_id}.txt
set -uo pipefail  # no -e: jq/security failures shouldn't crash silently

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
if [ -f "$COUNTER_FILE" ]; then
    _raw=$(cat "$COUNTER_FILE" 2>/dev/null || echo 0)
    # Validate numeric to avoid arithmetic crash on corrupted file
    [[ "$_raw" =~ ^[0-9]+$ ]] && COUNT=$_raw || COUNT=0
fi
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
{{< tab name="v1: Powerline" icon="eye" >}}

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

{{< callout type="info" >}}
The API usage part reads the OAuth token from macOS Keychain via `security find-generic-password`. On Linux, you'd need a different approach to read the token.
{{< /callout >}}

{{% details title="statusline.sh (v1)" closed="true" %}}

```bash
#!/usr/bin/env bash
set -uo pipefail  # no -e: external commands (git, kubectl, jq) can fail; silent crash = no statusline
trap 'printf "\n"' EXIT  # ensure at least empty output on crash
[ "${STATUSLINE_DEBUG:-}" = "1" ] && exec 2>/tmp/statusline-debug.log

DATA=$(timeout 2 cat 2>/dev/null) || DATA=""
[ -z "$DATA" ] && exit 0

# ── Extract ALL fields in a single jq call ──────────────────────────────────
# Uses jq @sh to produce shell-safe quoted assignments. No IFS tricks needed;
# empty fields become VAR='' instead of being silently swallowed.
eval "$(echo "$DATA" | jq -r '
    @sh "MODEL=\(.model.display_name // "Claude" | gsub(" \\(.*\\)"; ""))",
    @sh "MODEL_ID=\(try (.model.id // "unknown") catch "unknown")",
    @sh "DIR=\(.cwd // "~" | split("/") | .[-2:] | join("/"))",
    @sh "PCT=\(try (
        if (.context_window.remaining_percentage // null) != null then
            100 - (.context_window.remaining_percentage | floor)
        elif (.context_window.context_window_size // 0) > 0 then
            (((.context_window.current_usage.input_tokens // 0) +
              (.context_window.current_usage.cache_creation_input_tokens // 0) +
              (.context_window.current_usage.cache_read_input_tokens // 0)) * 100 /
             .context_window.context_window_size) | floor
        else 0 end
    ) catch 0)",
    @sh "CTX_SIZE=\(.context_window.context_window_size // 200000)",
    @sh "DURATION_MS=\(.cost.total_duration_ms // 0)",
    @sh "AGENT=\(.agent.name // "")",
    @sh "MODE=\(.mode // "")",
    @sh "TRANSCRIPT_PATH=\(.transcript_path // "")",
    @sh "CWD_FULL=\(.cwd // "~")",
    @sh "SESSION_ID=\(.session_id // "")",
    @sh "FIVE_PCT=\(.rate_limits.five_hour.used_percentage // "")",
    @sh "SEVEN_PCT=\(.rate_limits.seven_day.used_percentage // "")",
    @sh "FIVE_RESET_TS=\(.rate_limits.five_hour.resets_at // "")",
    @sh "SEVEN_RESET_TS=\(.rate_limits.seven_day.resets_at // "")"
' 2>/dev/null)" 2>/dev/null

# Guard: if jq failed completely, use safe defaults
MODEL=${MODEL:-Claude}; MODEL_ID=${MODEL_ID:-unknown}; DIR=${DIR:-~}
PCT=${PCT:-0}; CTX_SIZE=${CTX_SIZE:-200000}; DURATION_MS=${DURATION_MS:-0}
AGENT=${AGENT:-}; MODE=${MODE:-}; TRANSCRIPT_PATH=${TRANSCRIPT_PATH:-}
CWD_FULL=${CWD_FULL:-~}; SESSION_ID=${SESSION_ID:-}
FIVE_PCT=${FIVE_PCT:-}; SEVEN_PCT=${SEVEN_PCT:-}
FIVE_RESET_TS=${FIVE_RESET_TS:-}; SEVEN_RESET_TS=${SEVEN_RESET_TS:-}

PCT=${PCT%%.*}  # truncate jq float rounding (e.g. 14.000000000000002 -> 14)
FIVE_PCT=${FIVE_PCT%%.*}   # also truncate rate limit floats
SEVEN_PCT=${SEVEN_PCT%%.*}
CTX_SIZE_K=$((CTX_SIZE / 1000))
# Max line width before Claude Code's cli-truncate drops line 2
SAFE_WIDTH=${STATUSLINE_WIDTH:-110}

TOPIC=""  # populated after SESSION_ID is extracted below

# ── Effort level detection (transcript -> settings -> default) ──────────────
EFFORT=""
if [ -n "$TRANSCRIPT_PATH" ] && [ -f "$TRANSCRIPT_PATH" ]; then
    # Read from end of file for speed on large transcripts
    EFFORT=$(tail -r "$TRANSCRIPT_PATH" 2>/dev/null \
        | grep -m1 -E '"content":"<local-command-stdout>(Set model to.*effort|Set effort level to)' \
        | grep -oE '\b(low|medium|high|max)\b' | tail -1 || true)
fi
if [ -z "$EFFORT" ]; then
    EFFORT=$(jq -r '.effortLevel // empty' "$HOME/.claude/settings.json" 2>/dev/null || true)
fi
EFFORT=${EFFORT:-medium}

# ── Nerd Font icons ───────────────────────────────────────────────────────
NF_GIT=$'\xee\x82\xa0'       # U+E0A0 powerline branch
NF_FOLDER="󰉋"               # nf-md-folder (kept from v1)
NF_MODEL="󰚩"                # nf-md-robot (kept from v1)
NF_K8S="󱃾"                  # nf-md-kubernetes (kept from v1)
NF_CLOCK=$'\xef\x80\x97'     # U+F017 clock
NF_CORNER_TL=$'\xee\x82\xba'    # U+E0BA lower-right fill (top-left corner)
NF_CORNER_BL=$'\xee\x82\xbe'    # U+E0BE upper-right fill (bottom-left corner)
NF_CORNER_TR=$'\xee\x82\xb8'    # U+E0B8 lower-left fill -> top-right corner cut
NF_CORNER_BR=$'\xee\x82\xbc'    # U+E0BC upper-left fill -> bottom-right corner cut

# ── Project-colored background (hash session ID -> unique hue) ────────────
RST="\033[0m"
PROJECT_ROOT=$(git -C "$CWD_FULL" rev-parse --show-toplevel 2>/dev/null || echo "$CWD_FULL")
PHASH=$(printf '%s' "${SESSION_ID:-$CWD_FULL}" | cksum | cut -d' ' -f1 || echo "0")

# ── Session topic ─────────────────────────────────────────────────────────
if [ -n "${SESSION_ID:-}" ]; then
    TOPIC_FILE="$HOME/.claude/session-topics/${SESSION_ID}.txt"
    if [ -f "$TOPIC_FILE" ]; then
        # Strip ANSI escape sequences and limit to 40 chars
        TOPIC=$(cat "$TOPIC_FILE" 2>/dev/null | tr -d '\n' | gsed 's/\x1b\[[0-9;]*m//g' 2>/dev/null | cut -c1-40)
    fi
fi

# Check for manual color override
COLOR_OVERRIDES="$HOME/.claude/statusline-color-overrides.json"
if [ -f "$COLOR_OVERRIDES" ]; then
    COLOR_IDX=$(jq -r --arg p "$PROJECT_ROOT" '.[$p] // empty' "$COLOR_OVERRIDES" 2>/dev/null || true)
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
    local p=${1:-0}
    p=${p%%.*}  # safety: strip decimal if any
    if   [ "${p:-0}" -gt 80 ] 2>/dev/null; then printf "\033[38;2;225;150;150m"   # coral
    elif [ "${p:-0}" -gt 50 ] 2>/dev/null; then printf "\033[38;2;215;195;125m"   # gold
    else                                         printf "\033[38;2;150;210;150m"   # sage
    fi
}

# ── Git info ────────────────────────────────────────────────────────────────
BRANCH=$(git -c core.useBuiltinFSMonitor=false branch --show-current 2>/dev/null || echo "")
GIT_STATUS=""
if [ -n "$BRANCH" ]; then
  STAGED=$(git diff --cached --numstat 2>/dev/null | wc -l | tr -d " ")
  MODIFIED=$(git diff --numstat 2>/dev/null | wc -l | tr -d " ")
  UNTRACKED=$(git ls-files --others --exclude-standard 2>/dev/null | wc -l | tr -d " ")
  [ "${STAGED:-0}" -gt 0 ]    && GIT_STATUS="+${STAGED}"
  [ "${MODIFIED:-0}" -gt 0 ]  && GIT_STATUS="${GIT_STATUS:+$GIT_STATUS }!${MODIFIED}"
  [ "${UNTRACKED:-0}" -gt 0 ] && GIT_STATUS="${GIT_STATUS:+$GIT_STATUS }?${UNTRACKED}"
fi

# ── Kubernetes context (with timeout to avoid exec-auth hangs) ──────────────
K8S_CTX=$(timeout 2 kubectl config current-context 2>/dev/null || echo "")

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
    local pct=${1:-0} width=${2:-5} fill_clr="$3" empty_clr="$4" bar=""
    pct=${pct%%.*}  # safety: strip decimal
    [ -z "$pct" ] && pct=0
    local filled=$((pct * width / 100))
    [ "$pct" -gt 0 ] 2>/dev/null && [ "$filled" -eq 0 ] && filled=1
    [ "$filled" -gt "$width" ] && filled=$width
    [ "$filled" -lt 0 ] && filled=0
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

# ── Count visible columns (ANSI-aware, multi-string in one perl call) ──────
# Usage: measure_cols "str1" "str2" ... -> outputs one number per line
measure_cols() {
    local args=()
    for s in "$@"; do args+=("$(printf '%b' "$s")"); done
    printf '%s\n' "${args[@]}" | perl -ne '
        s/\e\[[0-9;]*m//g;
        chomp;
        use Encode qw(decode);
        my $decoded = decode("UTF-8", $_, Encode::FB_DEFAULT);
        print length($decoded), "\n";
    ' 2>/dev/null
}

# ── Line 1: build with bash-based width tracking (no perl) ─────────────────
# Estimate visible width per component from known text lengths.
# Each component's ANSI overhead cancels out; we just count visible chars.
# Corner char = 1, each separator "│" = 1, icon = 1-2, spaces counted explicitly.
# Add 3-char safety margin for Nerd Font icons that render wider than 1 codepoint.
L1_EST=5  # corner char(1) + trailing space(1) + icon safety margin(3)

[ -n "$TOPIC" ] && L1_EST=$((L1_EST + 1 + ${#TOPIC} + 3))   # " TOPIC │"
L1_EST=$((L1_EST + 1 + 2 + ${#DIR} + 1))                     # " icon DIR "
if [ -n "$BRANCH" ]; then
    L1_EST=$((L1_EST + 1 + 1 + 2 + ${#BRANCH}))              # "│ icon BRANCH"
    [ -n "$GIT_STATUS" ] && L1_EST=$((L1_EST + 1 + ${#GIT_STATUS}))
fi
[ -n "$AGENT" ] && L1_EST=$((L1_EST + 1 + ${#AGENT}))
[ -n "$MODE" ]  && L1_EST=$((L1_EST + 3 + ${#MODE}))         # " │ MODE"
[ -n "$K8S_CTX" ] && L1_EST=$((L1_EST + 1 + 1 + 2 + ${#K8S_CTX}))  # " │ icon K8S"

# Truncate if over SAFE_WIDTH (order: K8S > BRANCH > TOPIC)
if [ "$L1_EST" -gt "$SAFE_WIDTH" ] && [ -n "$K8S_CTX" ]; then
    OVER=$((L1_EST - SAFE_WIDTH))
    K8S_MAX=$((${#K8S_CTX} - OVER - 2))
    if [ "$K8S_MAX" -gt 5 ]; then
        K8S_CTX="${K8S_CTX:0:$K8S_MAX}.."
    else
        K8S_CTX=""
    fi
    # Recalculate
    L1_EST=2; [ -n "$TOPIC" ] && L1_EST=$((L1_EST + 1 + ${#TOPIC} + 3))
    L1_EST=$((L1_EST + 1 + 2 + ${#DIR} + 1))
    [ -n "$BRANCH" ] && { L1_EST=$((L1_EST + 1 + 1 + 2 + ${#BRANCH})); [ -n "$GIT_STATUS" ] && L1_EST=$((L1_EST + 1 + ${#GIT_STATUS})); }
    [ -n "$AGENT" ] && L1_EST=$((L1_EST + 1 + ${#AGENT}))
    [ -n "$MODE" ]  && L1_EST=$((L1_EST + 3 + ${#MODE}))
    [ -n "$K8S_CTX" ] && L1_EST=$((L1_EST + 1 + 1 + 2 + ${#K8S_CTX}))
fi

if [ "$L1_EST" -gt "$SAFE_WIDTH" ] && [ -n "$BRANCH" ]; then
    OVER=$((L1_EST - SAFE_WIDTH))
    BR_MAX=$((${#BRANCH} - OVER - 2))
    if [ "$BR_MAX" -gt 5 ]; then
        BRANCH="${BRANCH:0:$BR_MAX}.."
    else
        BRANCH="${BRANCH:0:8}.."
    fi
    # No need to recalculate again; next truncation target (TOPIC) is rare
fi

if [ "$L1_EST" -gt "$SAFE_WIDTH" ] && [ -n "$TOPIC" ]; then
    # Recalculate after branch truncation
    L1_EST=2; [ -n "$TOPIC" ] && L1_EST=$((L1_EST + 1 + ${#TOPIC} + 3))
    L1_EST=$((L1_EST + 1 + 2 + ${#DIR} + 1))
    [ -n "$BRANCH" ] && { L1_EST=$((L1_EST + 1 + 1 + 2 + ${#BRANCH})); [ -n "$GIT_STATUS" ] && L1_EST=$((L1_EST + 1 + ${#GIT_STATUS})); }
    [ -n "$AGENT" ] && L1_EST=$((L1_EST + 1 + ${#AGENT}))
    [ -n "$MODE" ]  && L1_EST=$((L1_EST + 3 + ${#MODE}))
    [ -n "$K8S_CTX" ] && L1_EST=$((L1_EST + 1 + 1 + 2 + ${#K8S_CTX}))
    if [ "$L1_EST" -gt "$SAFE_WIDTH" ]; then
        OVER=$((L1_EST - SAFE_WIDTH))
        T_MAX=$((${#TOPIC} - OVER - 2))
        if [ "$T_MAX" -gt 5 ]; then
            TOPIC="${TOPIC:0:$T_MAX}.."
        else
            TOPIC=""
        fi
    fi
fi

# Assemble Line 1 from (possibly truncated) components
L1_PREFIX="${RST}${PROJ_FG}${NF_CORNER_TL}${BG1}"
L1C="${L1_PREFIX}"
[ -n "$TOPIC" ] && L1C+=" ${TXT_BOLD}${TOPIC}${B} ${SEP}${B}"
L1C+=" ${TXT_FG}${NF_FOLDER} ${DIR} ${B}"
if [ -n "$BRANCH" ]; then
    L1C+="${SEP}${B} ${TXT_FG}${NF_GIT} ${BRANCH}${B}"
    [ -n "$GIT_STATUS" ] && L1C+=" ${TXT_FG}${GIT_STATUS}${B}"
fi
[ -n "$AGENT" ] && L1C+=" ${TXT_FG}${AGENT}${B}"
[ -n "$MODE" ]  && L1C+=" ${SEP}${B} \033[1;38;2;150;100;0m${MODE}${B}"
[ -n "$K8S_CTX" ] && L1C+=" ${SEP}${B} ${TXT_FG}${NF_K8S} ${K8S_CTX}${B}"
L1C+=" "

# ── Line 2 content ──────────────────────────────────────────────────────────
CTX_CLR=$(pct_txt_color "$PCT")
CTX_BAR=$(make_bar "$PCT" 7 "$CTX_CLR" "$L2_DIM")
# Effort level color
case $EFFORT in
    max)    EFFORT_CLR="\033[38;2;150;210;150m" ;;  # sage
    high)   EFFORT_CLR="\033[38;2;150;210;150m" ;;  # sage
    medium) EFFORT_CLR="\033[38;2;170;170;170m" ;;  # gray (blends in)
    low)    EFFORT_CLR="\033[38;2;225;150;150m" ;;  # coral (warning)
    *)      EFFORT_CLR="\033[38;2;170;170;170m" ;;  # fallback: gray
esac

L2C="${RST}\033[38;2;0;0;0m${NF_CORNER_BL}${BG2} ${L2_TXT}${NF_MODEL} ${MODEL} ${L2_DIM}·${B2} ${EFFORT_CLR}${EFFORT}${B2} ${L2_DIM}│${B2} ${L2_TXT}${NF_CLOCK} ${TIME_CLR}${TIME}${B2} ${L2_DIM}│${B2} ${CTX_BAR} ${CTX_CLR}${PCT}%${B2} ${L2_TXT}of ${CTX_SIZE_K}k"

# Estimate L2 base width with bash (same approach as L1: count visible chars)
# Model + effort + separator + clock + time + separator + bar(7) + pct + "of Xk"
L2_BASE_W=$((2 + 2 + ${#MODEL} + 1 + ${#EFFORT} + 3 + 2 + ${#TIME} + 3 + 7 + 1 + ${#PCT} + 1 + 4 + ${#CTX_SIZE_K} + 1))
L2_BASE_W=${L2_BASE_W:-50}
RATE_AVAIL=$((SAFE_WIDTH - L2_BASE_W - 5))   # reserve 5 for incident icon

if [ -n "${FIVE_PCT:-}" ] && [ -n "${SEVEN_PCT:-}" ]; then
    FIVE_CLR=$(pct_txt_color "$FIVE_PCT")
    SEVEN_CLR=$(pct_txt_color "$SEVEN_PCT")

    if [ "$RATE_AVAIL" -gt 40 ] 2>/dev/null; then
        # Full: bars + pct + reset times
        FIVE_BAR=$(make_bar "$FIVE_PCT" 5 "$FIVE_CLR" "$L2_DIM")
        FIVE_TIME=$(format_reset "$FIVE_RESET_TS")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}5h ${FIVE_BAR} ${FIVE_CLR}${FIVE_PCT}%${B2}"
        [ -n "${FIVE_TIME:-}" ] && L2C+=" ${L2_DIM}${FIVE_TIME}${B2}"
        SEVEN_BAR=$(make_bar "$SEVEN_PCT" 5 "$SEVEN_CLR" "$L2_DIM")
        SEVEN_TIME=$(format_reset "$SEVEN_RESET_TS")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}7d ${SEVEN_BAR} ${SEVEN_CLR}${SEVEN_PCT}%${B2}"
        [ -n "${SEVEN_TIME:-}" ] && L2C+=" ${L2_DIM}${SEVEN_TIME}${B2}"
    elif [ "$RATE_AVAIL" -gt 25 ] 2>/dev/null; then
        # Compact: bars + pct, no reset times
        FIVE_BAR=$(make_bar "$FIVE_PCT" 5 "$FIVE_CLR" "$L2_DIM")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}5h ${FIVE_BAR} ${FIVE_CLR}${FIVE_PCT}%${B2}"
        SEVEN_BAR=$(make_bar "$SEVEN_PCT" 5 "$SEVEN_CLR" "$L2_DIM")
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}7d ${SEVEN_BAR} ${SEVEN_CLR}${SEVEN_PCT}%${B2}"
    elif [ "$RATE_AVAIL" -gt 15 ] 2>/dev/null; then
        # Minimal: just percentages
        L2C+=" ${L2_DIM}│${B2} ${L2_TXT}5h ${FIVE_CLR}${FIVE_PCT}%${B2} ${L2_TXT}7d ${SEVEN_CLR}${SEVEN_PCT}%${B2}"
    fi
fi

# ── Claude service status (auto-refresh every 60s in background) ────────────
SVC_CACHE="/tmp/claude-service-status"
SVC_FETCH="$HOME/.claude/claude-status-fetch.sh"
if [ -x "$SVC_FETCH" ]; then
    SVC_AGE=9999
    [ -f "$SVC_CACHE" ] && SVC_AGE=$(($(date +%s) - $(stat -f %m "$SVC_CACHE" 2>/dev/null || echo 0)))
    if [ "$SVC_AGE" -ge 60 ]; then
        ("$SVC_FETCH" >/dev/null 2>/dev/null &)
    fi
fi
if [ -f "$SVC_CACHE" ]; then
    SVC_RAW=$(head -1 "$SVC_CACHE" 2>/dev/null)
    case "${SVC_RAW:-}" in
        operational)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;100;200;120m✓${B2}"
            ;;
        incident:*)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;225;150;100m⚠${B2}"
            ;;
        degraded_performance:*)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;215;195;125m~${B2}"
            ;;
        partial_outage:*|major_outage:*)
            L2C+=" ${L2_DIM}│${B2} \033[38;2;225;100;100m✗${B2}"
            ;;
    esac
fi

L2C+=" "

# ── Set terminal tab title ───────────────────────────────────────────────────
_TAB_TITLE="${TOPIC:-${DIR:-Claude}}"
printf '\033]1;%s\007' "$_TAB_TITLE" > /dev/tty 2>/dev/null || true

# ── Pad shorter line to match longer ────────────────────────────────────────
{
    # Single perl invocation for both line measurements
    read -r L1_COLS L2_COLS < <(
        measure_cols "$L1C" "$L2C" | tr '\n' ' '
    )
    L1_COLS=${L1_COLS:-0}; L2_COLS=${L2_COLS:-0}
    SYNC_W=$L2_COLS
    [ "$L1_COLS" -gt "$SYNC_W" ] 2>/dev/null && SYNC_W=$L1_COLS
    if [ "$L1_COLS" -gt 10 ] 2>/dev/null && [ "$L1_COLS" -lt "$SYNC_W" ] 2>/dev/null; then
        L1C+="${BG1}$(printf '%*s' "$((SYNC_W - L1_COLS))" '')"
    fi
    if [ "$L2_COLS" -gt 10 ] 2>/dev/null && [ "$L2_COLS" -lt "$SYNC_W" ] 2>/dev/null; then
        L2C+="${BG2}$(printf '%*s' "$((SYNC_W - L2_COLS))" '')"
    fi
} 2>/dev/null || true

# ── Output ───────────────────────────────────────────────────────────────────
L2_END_FG="\033[38;2;0;0;0m"
trap - EXIT  # disarm crash trap before normal output
printf '\033[0m%b\n' "${L1C}${RST}${PROJ_FG}${NF_CORNER_TR}${RST}"
printf '\033[0m%b\n' "${L2C}${RST}${L2_END_FG}${NF_CORNER_BR}${RST}"
```

{{% /details %}}

{{< /tab >}}
{{< /tabs >}}

## Adapting It

Each segment is independent. Ask Claude Code to adapt the script to your needs, or use it as a starting point for your own design.

