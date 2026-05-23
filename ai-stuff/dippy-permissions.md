# Supercharging Claude Code Permissions with Dippy

{{< callout type="info" >}}
**TL;DR:** Native Claude Code permissions don't do guidance messages, last-match-wins ordering, or command delegation. Dippy (a `PreToolUse` hook) does. Claude Code later shipped auto-mode, an LLM permission classifier designed to eliminate prompts, and a thin wrapper now runs Dippy inside auto-mode too. Explicit `allow` / `deny` rules short-circuit the classifier; an `[ASK]` marker on the scary stuff (force-push, `rm -rf`, `tofu apply`, etc.) still prompts me even when I'm hands-off. Everything else falls through to auto-mode. One config, both modes.
{{< /callout >}}

Claude Code asks for permission before running shell commands and MCP tools. Out of the box, you manage this through `settings.json`, a flat JSON list of `allow` and `deny` entries. My original frustration was wildcards: I wanted to write `Bash(kubectl -n * get)` to match any namespace, but at the time I thought settings.json only supported prefix matching. No way to put a `*` in the middle of a command.

That's what first pushed me toward Dippy. While writing this post I noticed Claude Code actually added wildcard support at any position back in [v2.1.0](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md#210) (January 2026), so `Bash(kubectl -n * get)` works natively today. The wildcard gap closed, but the rest of the gap (guidance messages, last-match-wins, file-redirect controls, command delegation) is still there. That's why I kept Dippy.

## What is Dippy

<img src="/images/dippy.gif" alt="Dippy demo" style="max-width: 350px;" />

[🐤 Dippy](https://github.com/ldayton/Dippy) is a CLI tool that runs as a [Claude Code hook](https://docs.anthropic.com/en/docs/claude-code/hooks). It intercepts `PreToolUse` events (before Claude executes a Bash command or MCP tool) and decides: allow, ask (prompt with a message), or deny.

The config is a plain text file with pattern matching:

```bash
allow git status        # auto-approve
ask git push "Confirm push target"  # prompt with message
deny rm -rf "Use trash instead"     # block with explanation
```

What makes it better than `settings.json`:

1. **Guidance messages** - settings.json has allow, ask, and deny, but the prompts are bare. Dippy lets you attach a message: `deny helm install "Use GitOps: create a HelmRelease in the k8s/flux repo instead"` teaches Claude the right approach without me explaining it every time
2. **Last match wins** - rules are ordered and the last match takes precedence, in any direction. In settings.json, deny always wins over allow regardless of order, so you can't do "deny git push, but allow git push origin main". Dippy's ordering gives you full control
3. **Plain text config** - much easier to organize, read, and maintain than a JSON array. Comments, categories, blank lines for grouping
4. **File redirect controls** - `deny-redirect **/.env* "Never write secrets"` blocks output redirection to sensitive files, even if the command itself is allowed
5. **Command delegation** - Dippy can trace into nested commands. For example, `kubectl exec pod -- cat /etc/config` is evaluated as two decisions: the outer `kubectl exec` and the inner `cat /etc/config` (with `remote=True` so local path checks are skipped). Native permissions only see the full command string and can't match rules against just the inner command after `--`

## The Config

Dippy's config reads like documentation. Here's how I organized mine:

### Default Behavior

```bash
set default ask
```

Unknown commands prompt for approval rather than silently blocking. This is important: I want to know when Claude tries something new, not have it fail silently.

### File Redirect Controls

```bash
deny-redirect **/.env* "Never write secrets; do it manually"
deny-redirect **/*credentials* "Never write credentials; do it manually"
deny-redirect **/*.pem "Never write key files; do it manually"
deny-redirect **/id_rsa* "Never write SSH keys; do it manually"
```

This catches output redirection. Even if a command is allowed, writing its output to a secrets file gets blocked with a clear message.

### Read vs Write Separation

The biggest improvement over settings.json is separating read-only operations from write operations. Git is a good example:

```bash
# Git read operations - auto-approve
allow git status
allow git log
allow git diff
allow git show
allow git fetch
allow git branch
allow git blame

# Git write operations - prompt first, except pull
allow git pull
ask git push "Confirm push target"
ask git commit "Confirm commit"
ask git add "Confirm staging"

# Destructive git operations - prompt with warning
ask git push --force "WARNING: force-push can destroy remote history"
ask git reset --hard "WARNING: discards uncommitted changes permanently"
ask git branch -D "WARNING: force-deletes branch; consider git branch -d"
```

"Last match wins" makes this work. `allow git` would match `git push`, but the more specific `ask git push` rule comes later and takes precedence.

I use `ask` instead of `deny` for destructive operations. The warning message shows me why I should be careful, but I can still approve it when I actually want it. With `deny`, Claude would just give up and suggest I do it manually.

### Kubernetes, Helm, Flux

Same pattern: read operations are auto-approved, write operations prompt, and some are denied with guidance:

```bash
# Read - auto-approve
allow kubectl get
allow kubectl describe
allow kubectl logs
allow helm list
allow flux get

# Write - prompt
ask kubectl apply "Confirm resource apply"
ask kubectl delete "WARNING: will delete cluster resources"
deny helm install "Use GitOps: create a HelmRelease in the k8s/flux repo instead"
deny helm uninstall "Use GitOps: remove the HelmRelease from the k8s/flux repo instead"
```

### GitHub CLI

The same read/write pattern works for API tools. Viewing and listing are safe, but creating issues, closing them, or posting comments are visible to others and shouldn't happen silently:

```bash
# Read - auto-approve
allow gh issue view
allow gh issue list
allow gh pr view
allow gh search

# Write - prompt (creates visible artifacts)
ask gh issue create "Confirm issue creation"
ask gh issue close "Confirm issue close"
ask gh issue comment "Confirm posting comment"
ask gh pr create "Confirm PR creation"
ask gh pr comment "Confirm posting PR comment"
```

`gh api` gets the same curl-style treatment: GET requests are allowed, but write methods prompt:

```bash
allow gh api
ask gh api * -X POST "Review POST request before sending"
ask gh api * -X DELETE "Review DELETE request before sending"
```

Unlisted `gh` commands (like `gh repo delete` or `gh pr merge`) fall through to `set default ask`, so they still prompt. No broad `allow gh` needed.

### Network Requests

Curl is interesting because `curl -s` (read) and `curl -X POST` (write) are very different operations:

```bash
allow curl
# Write operations - catch flags regardless of position (last match wins)
ask curl * -X POST "Review POST request before sending"
ask curl * -X PUT "Review PUT request before sending"
ask curl * -X DELETE "Review DELETE request before sending"
ask curl * -X PATCH "Review PATCH request before sending"
ask curl * -d "Review request with data before sending"
ask curl * -F "Review form upload before sending"
ask curl * -T "Review file upload before sending"
```

### MCP Tools

Dippy also handles MCP tool permissions. I added a second hook matcher for `mcp__.*` and moved all MCP rules out of settings.json:

```bash
# Documentation tools
allow-mcp mcp__context7__resolve-library-id
allow-mcp mcp__context7__query-docs

# Monitoring (read operations + ask for writes)
allow-mcp mcp__prometheus__*
allow-mcp mcp__grafana__*
ask-mcp mcp__grafana__update_dashboard "Confirm dashboard update"

# Kubernetes MCP
allow-mcp mcp__kubernetes__resources_get
allow-mcp mcp__kubernetes__resources_list
allow-mcp mcp__kubernetes__pods_list
```

## Per-Project Overrides

The global config covers common tools. Project-specific rules go in `.dippy` files at the repo root. Dippy's precedence: project `.dippy` overrides global `~/.dippy/config`.

I created 8 project `.dippy` files for repos that need extra permissions. For example, my Home Assistant repo needs access to the Home Assistant MCP server, which doesn't make sense globally:

```bash
# hasspi/.dippy
allow-mcp mcp__hass-mcp__*
```

Or my Flux monorepo, which needs a project-specific Harbor API script and the Flux operator MCP:

```bash
# k8s/flux/.dippy
allow /path/to/harbor-api.py exec
allow-mcp mcp__flux-operator-mcp__*
```

## Coexisting with auto-mode

Claude Code's auto-mode is an LLM permission classifier with three rule buckets in `settings.json`: `allow`, `soft_deny`, and `hard_deny`. It's designed to eliminate prompts entirely. Perfect for unattended runs, hostile to Dippy's guidance messages, at least at first glance.

The leverage point is timing. `PreToolUse` hooks fire **before** the permission system per the hook lifecycle, including in `auto`, which means a Dippy decision can short-circuit the classifier entirely whenever I want it to. The question is which decisions to forward and which to let the classifier handle. My current wrapper threads that needle:

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/dippy-with-auto-fallback.sh
set -u
payload=$(cat)
mode=$(printf '%s' "$payload" | jq -r '.permission_mode // "default"' 2>/dev/null)

if [ "$mode" = "auto" ]; then
  output=$(printf '%s' "$payload" | dippy)
  decision=$(printf '%s' "$output" | jq -r '.hookSpecificOutput.permissionDecision // ""' 2>/dev/null)
  reason=$(printf '%s' "$output" | jq -r '.hookSpecificOutput.permissionDecisionReason // ""' 2>/dev/null)
  case "$decision" in
    allow|deny) printf '%s' "$output" ;;
    ask)
      case "$reason" in
        *"[ASK]"*) printf '%s' "$output" ;;
        *)         : ;;
      esac
      ;;
    *) : ;;
  esac
  exit 0
fi

printf '%s' "$payload" | dippy
```

In `default` / `acceptEdits` / `plan` / `bypassPermissions` modes, Dippy runs unchanged. In `auto`, the wrapper runs Dippy first and only forwards its output to Claude Code when:

- The decision is `allow`: short-circuit, approve instantly, no classifier roundtrip
- The decision is `deny`: short-circuit, block with Dippy's guidance message
- The decision is `ask` **and** the rule's message contains the literal string `[ASK]`: force a prompt even in auto-mode

Anything else (unknown commands that hit `set default ask`, or `ask` rules without the marker) gets swallowed and the classifier decides as normal.

### The [ASK] marker

Auto-mode's whole pitch is "no prompts." Forwarding every `ask` decision would defeat it. But some commands are scary enough that I want to be in the loop even when I'm not paying close attention. The marker lets me opt those in explicitly:

```bash
ask git push "[ASK] Confirm push target"
ask git push --force "[ASK] WARNING: force-push can destroy remote history"
ask git reset --hard "[ASK] WARNING: discards uncommitted changes permanently"
ask git rebase "[ASK] WARNING: rebase rewrites history; confirm target branch"
ask kubectl delete "[ASK] WARNING: will delete cluster resources"
ask kubectl apply "[ASK] Confirm resource apply"
ask kubectl drain "[ASK] WARNING: will evict all pods from node"
ask tofu apply "[ASK] Confirm tofu apply"
ask tofu destroy "[ASK] WARNING: will destroy infrastructure"
ask rm -rf "[ASK] WARNING: recursive delete"
ask sudo "[ASK] WARNING: running as root"
ask gh pr create "[ASK] Confirm PR creation"
ask gh issue comment "[ASK] Confirm posting comment"
```

The pattern I converged on: anything that touches remote state visible to other humans (`git push`, `gh pr create`, `gh issue comment`), anything that rewrites or destroys git history, anything that mutates a production cluster, anything that runs as root, anything that recursively deletes. About 28 rules in total. Everything else falls through to the classifier.

`deny` rules don't need a marker. They always short-circuit in auto-mode under this wrapper, so the GitOps-style guidance still reaches the model:

```bash
deny helm install "Use GitOps: create a HelmRelease in the k8s/flux repo instead"
deny helm upgrade "Use GitOps: update the HelmRelease in the k8s/flux repo instead"
deny helm uninstall "Use GitOps: remove the HelmRelease from the k8s/flux repo instead"
```

### Wiring it in

Swap the `"command"` in `settings.json` for the wrapper path on both Bash and MCP matchers:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "/Users/me/.claude/hooks/dippy-with-auto-fallback.sh" }
        ]
      },
      {
        "matcher": "mcp__.*",
        "hooks": [
          { "type": "command", "command": "/Users/me/.claude/hooks/dippy-with-auto-fallback.sh" }
        ]
      }
    ]
  }
}
```

### Defense in depth (optional)

Since the wrapper now forwards Dippy's denies in auto-mode, the native `permissions.deny` mirror isn't load-bearing anymore. I keep a slim version anyway. If something ever bypasses the hook (a future Claude Code release, a hook crash, a renamed script), the must-hold rules still hold:

```json
{
  "permissions": {
    "deny": [
      "Bash(git push --force*)",
      "Bash(kubectl delete*)",
      "Bash(rm -rf *)"
    ]
  },
  "autoMode": {
    "hard_deny": [
      "$defaults",
      "Never run helm install or helm uninstall: use GitOps via the k8s/flux repo instead"
    ]
  }
}
```

Two layers below Dippy means the dangerous stuff stops even if the hook doesn't.

### Net result

The split now tracks intent rather than mode. Things I always want to be in the loop on (irreversible, public, or root) get the `[ASK]` marker and prompt me even in auto-mode. Things Dippy already has a policy for (explicit allow or deny rules) short-circuit the classifier. Everything else flows through auto-mode at full speed.

## The Migration from settings.json to Dippy

I pointed Claude at my global `settings.json` (191 entries) and 14 project-local `settings.local.json` files, and asked it to categorize everything and migrate it all to Dippy configs.

The process went roughly like this:

1. **Audit** - Claude read every settings file, categorized all 191 entries by tool type (git, kubectl, helm, docker, etc.), and identified duplicates across repos
2. **Create global Dippy config** - Translated all Bash and MCP permissions into Dippy format, organized by category, with read/write separation
3. **Slim down settings.json** - Removed all `Bash(...)` and `mcp__*` entries, keeping only Read, WebFetch, and Skill permissions (191 → 4 entries)
4. **Migrate project-local permissions** - For each repo, decided what to promote to global vs keep project-specific. Created 8 project `.dippy` files
5. **Add MCP hook** - Added a second PreToolUse matcher for `mcp__.*` so Dippy intercepts MCP tools too
6. **Validate** - Tested that all previously allowed commands still work, no new prompts for safe operations
7. **Create a [skill]({{< relref "commands-vs-mcp-vs-skills" >}})** - Asked Claude to build a skill for managing Dippy permissions going forward. Now when I need to allow a new command, I just say `/claude-permissions allow devbox run -- hugo server -D` and Claude handles the config edit. I review the diff, not write it

## Quick Start

Install Dippy:

```bash
brew tap ldayton/dippy
brew install dippy
```

Create a config at `~/.dippy/config`:

```bash
set default ask

allow git status
allow git log
allow git diff
allow kubectl get
allow kubectl describe
ask git push "Confirm push target"
deny rm -rf "Use trash instead"
```

Add the hook to your Claude Code `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "dippy" }
        ]
      }
    ]
  }
}
```

That's it. Every Bash command Claude runs now goes through Dippy first.

## Results

The day-to-day workflow is noticeably better:

- **No more permission fatigue** - safe commands auto-approve, dangerous ones prompt with context
- **Self-documenting** - the config file has comments and categories; `settings.json` was just a wall of strings
- **Guidance over blocking** - when Claude tries `helm install`, it sees "Use GitOps: create a HelmRelease in the k8s/flux repo instead" rather than a bare denial and me having to explain it should do GitOps

Pairing this with auto-mode through the wrapper gives me one setup that scales from "watch every command" to "just go", without rewriting policy in between. The same config governs both modes; the wrapper decides what to enforce silently, what to surface, and what to delegate to the classifier.
