# Claude, Codex, and a skipper

<img src="/images/claude-codex-skipper.png" alt="A robot ship captain holding a status tablet stands next to me on a night-time ship bridge, while pairs of orange and blue robots work at consoles and send question bubbles toward the captain" class="hero-image" style="max-width: 700px; width: 100%; height: auto;" />

{{< callout type="info" >}}
**TL;DR:** I want Claude and Codex on the same task, but I only like one harness: Claude Code. **clodex** puts Codex (GPT) models inside Claude Code, as the main model or as subagents, and **session-peers** gives every Claude session a real Codex CLI "buddy" that implements, reviews and argues back. Once I had ten of those pairs running, I lost track of who was doing what, so I built **skipper**: one session that keeps the roster, relays every question meant for me, and flags when a pair needs a handover. I talk to skipper and only open a pair's terminal when I want to dig in.
{{< /callout >}}

## Why two models at all

Claude and Codex fail differently. When one writes a plan, the other finds the holes in it; when one writes a diff, the other reviews it without the author's blind spots. In my experience, a second opinion from the same model family tends to agree with itself.

The catch is the harness (the CLI that wraps the model with tools, prompts and rules). I live in Claude Code: [skills]({{< ref "ai-stuff/commands-vs-mcp-vs-skills" >}}), hooks, [permission rules]({{< ref "ai-stuff/dippy-permissions" >}}), [statusline]({{< ref "ai-stuff/claude-statusline" >}}), subagents, all tuned over months. The Codex CLI is fine, I just do not enjoy driving it. So the question became: how do I get Codex's brain without moving into Codex's house?

I ended up with two answers and use both, plus a third piece once there were too many pairs to follow:

| | clodex | session-peers buddy | skipper |
|---|---|---|---|
| What you get | Codex models inside Claude Code | A real Codex CLI thread next to each Claude session | One session coordinating many pairs |
| Context | Shares Claude Code's harness and system prompt | Its own session and context | Tracks pairs, never reads their code |
| Best for | A quick second opinion, a GPT subagent | Long tasks: implement, review, argue back | Three or more pairs at once |
| Start here if | You want to try the other model today | You want a review partner that remembers | You keep asking terminals "where are we?" |

## Option 1: Codex models inside Claude Code (clodex)

Claude Code has no OpenAI provider. What it does support is pointing `ANTHROPIC_BASE_URL` at any gateway that speaks the Anthropic Messages API, so there is a small zoo of proxies that translate to OpenAI over your ChatGPT login. I tried one of them (opencodex) first. It worked, GPT models showed up in `/model`, but the Claude Code version I was on capped those gateway models at a 200k auto-compact window, no matter how much context the proxy advertised. So I reverted and kept looking.

What stuck is [clodex](https://github.com/bman654/clodex). It bridges Claude Code to OpenAI models through your ChatGPT/Codex plan (or an API key). In its default proxy mode, Claude Code keeps its own Anthropic login; only requests for a `clodex:` model get routed to OpenAI. Its `patch` step bakes your favorite models into the Claude Code binary so they show up in `/model`, report their real context window, and work as subagents.

```bash
npm install -g @bman654/clodex                          # Node 22+
clodex providers auth openai                            # device-code sign-in with your ChatGPT/Codex plan
clodex models                                           # interactive: pick favorites (max 20)
clodex models --alias sol=clodex:openai-oauth:gpt-6-sol # short name for /model and frontmatter
clodex patch                                            # make favorites first-class in Claude Code
clodex claude                                           # launch Claude Code through the bridge
```

After that, `/model sol` switches the whole session to GPT, and the model is also a valid `model:` for the Agent tool. That second part is what I actually use most. A few prompts straight from my history:

```text
for 1404 gpt-5.6-sol model spin up agent to review it pls, dont wait on CR
when u trigger future agents trigger them on gpt-5.6-sol model pls
```

**What to know before you rely on it:**

- It patches the Claude Code binary. Re-run `clodex patch` after **every** Claude Code update (and every clodex upgrade or favorite change), then restart Claude Code. A `FAIL` line in the patch output usually means clodex has not caught up with the newest Claude Code build yet.
- A new model missing from `clodex models`? The model list is cached and upgrading clodex does not refresh it: run `clodex providers refresh-models openai-oauth` first.
- Mind the context window. `clodex models --context <model>=max --save` opts into the model's current ceiling, which can cross a [pricing boundary](https://github.com/bman654/clodex#context-stops-and-the-pricing-boundary) where the whole request is billed at a higher rate. Where that boundary sits, and whether `standard` stays under it, depends on the model and provider, so check `clodex models --json` for the resolved window and `pricingBoundary` first. After changing it, re-run `clodex patch` and restart Claude Code.
- Do not combine it with another binary patcher such as tweakcc. Both restore their own "pristine" copy before patching, so each run wipes the other's changes.
- It works most of the time.
- It feels slow to me, although Codex models generally do. I have not measured it, so I do not blame the bridge.
- The GPT model runs inside Claude Code's harness and system prompt. A different model family is a real second opinion, but it is not a fully independent reviewer: it sees the world through the same tools and instructions.
- For long work I prefer a separate session I can talk to directly, in its own terminal and its own harness. That is the next option.

I keep the install, sign-in, add-a-model and re-patch steps in a small [clodex skill](https://github.com/vtmocanu/skills/tree/main/skills/clodex) so `/clodex gpt-6.1-sol` does the whole dance for a new model.

### Just started trying: claudish

[claudish](https://github.com/MadAppGang/claudish) (MIT, from MadAppGang) comes at the same problem from the other side. It does not patch the Claude Code binary. It starts a local proxy and launches Claude Code pointed at it, so the main conversation runs on whatever model you pick: your ChatGPT/Codex plan, OpenRouter, Gemini, or a local model.

```bash
brew tap MadAppGang/tap && brew install claudish       # or: npm install -g claudish
claudish login codex                                   # browser sign-in with your ChatGPT plan, no API key
claudish --probe cx@gpt-6.1-sol                        # check which credential will serve this model
```

My full `cc` command for GPT with auto-mode safety checks:

```bash
CLAUDE_CODE_MAX_CONTEXT_TOKENS=872000 \
CLAUDE_CODE_AUTO_COMPACT_WINDOW=784800 \
claudish \
  --model cx@gpt-6.1-sol \
  --no-auto-approve \
  --classifier-provider anthropic \
  --permission-mode auto
```

- `--no-auto-approve` prevents bypass; the [classifier](https://github.com/MadAppGang/claudish/blob/v14.1.0/docs/models/model-mapping.md#auto-mode-classifier-passthrough) can use my Claude subscription login when no `ANTHROPIC_API_KEY` is supplied.
- Window sizes are specific to my setup; check your provider's metadata before copying them.

Too early for a verdict. The appeal is no re-patching after every Claude Code update. The thing I use clodex for most is a GPT model as a subagent inside an otherwise-Claude session. claudish has a `--model-subagent` flag (one model for all subagents) and keeps your Claude login when a role maps to a Claude model, so that looks possible, but I have not tested it yet.

#### Free OpenRouter models, for a one-time $10

claudish also talks to [OpenRouter](https://openrouter.ai/), which has a set of free models (IDs ending in `:free`). Free is not unlimited. Per the [OpenRouter limits docs](https://openrouter.ai/docs/api-reference/limits), free models are capped at 20 requests a minute, and at 50 a day until you have bought at least $10 of credits, then 1,000 a day. The threshold counts all-time purchases, not your current balance, so one $10 top-up unlocks it for good.

My setup:

1. Create an OpenRouter account and an API key.
2. Buy $10 of credits once. Auto top-up was off by default for me; leave it off.
3. Add a workspace budget so the $10 is also a hard ceiling. I set a **Lifetime** budget of $10, so OpenRouter blocks further requests once I have spent what I put in.

   ![OpenRouter Budgets & data settings: a lifetime workspace budget of $10, current spend $0.00 of $10.00](/images/openrouter-workspace-budget.png)

4. Give claudish the key, either as an env var or in the macOS Keychain:

   ```bash
   export OPENROUTER_API_KEY=sk-or-v1-...
   claudish keychain set OPENROUTER_API_KEY   # alternative: prompts for the key, keeps it out of dotfiles
   ```

5. Pick a free model. There are two ways (`--free` is in the help text, but its picker comes back empty in claudish 14.1.0):

   ```bash
   claudish --model openrouter@openrouter/free                          # OpenRouter picks a free model per request
   claudish --model openrouter@nvidia/nemotron-3-ultra-550b-a55b:free   # pin one model
   ```

   `openrouter/free` is the zero-effort option, but it can switch models between turns (my first two requests went to two different ones). For a steadier session, pin one. Claude Code only works with models that support tool calls, so filter for those:

   ```bash
   curl -s https://openrouter.ai/api/v1/models \
     | jq -r '.data[] | select(.id|endswith(":free")) | select(.supported_parameters|index("tools")) | .id'
   ```

Two caveats. 1,000 a day is less than it sounds for an agent: every agent turn, so every tool call, is a request, and a busy Claude Code session burns through them fast and hits the per-minute cap first. And some providers may train on or retain prompts, so check the free-model training settings in [OpenRouter's privacy settings](https://openrouter.ai/settings/privacy) before pointing it at code you care about.

## Option 2: a real Codex buddy (session-peers)

For anything longer than one review, I want a second **session**, not a subagent: a Codex CLI thread with its own context that can implement for an hour, rerun the gate, and still remember what we agreed. That is what [session-peers](https://github.com/vtmocanu/skills/tree/main/skills/session-peers) does. It lets Claude Code sessions and Codex threads on the same machine message each other, with a small "shim" per Codex thread that delivers incoming messages as its next turn and forwards its replies back.

**Setup, once:**

```bash
npx -y skills@latest add vtmocanu/skills --skill session-peers -a claude-code codex -g -y
~/.agents/skills/session-peers/scripts/peers.py install-hook --auto-attach
```

Then open `/hooks` in Codex and trust the new entry. From then on every Codex thread you start or resume attaches automatically and shows up as a peer of your Claude sessions. Without the hook, attach a thread by hand with `~/.agents/skills/session-peers/scripts/peers.py up <name|uuid>`. The same script's `list` shows every live peer, and `doctor` explains a missing one.

**Per task:** I start a Claude session and a Codex thread and give both the same name (`/rename land90` in both), then tell Claude:

```text
buddy: land90, you lead
```

That binds the Codex thread as Claude's buddy. Binding does not decide who leads, so I say it (if I don't, the skill asks once and recommends Claude). Then "ping buddy": a pong back means the route works both ways. Over time my binding prompts got shorter: from "u ask @review codex session to review it pls and do ping/pong with it" to "ur buddy: web" to just "buddy".

### Who does what

The split that stuck:

| | Claude (lead) | Codex (buddy) |
|---|---|---|
| Talks to me | yes | rarely |
| Plans, brainstorms | yes, with the buddy | yes, with the lead |
| Writes code | small fixes | the heavy lifting, long test runs, rebases |
| Reviews | the buddy's diffs | the lead's diffs |
| Decides merges | within my grant | no |

Two rules make it work:

- **Nobody reviews their own diff.** The lead reviews what Codex wrote; Codex reviews what the lead wrote.
- **Verdicts are explicit and pinned.** A review starts with `APPROVE`, `REVISE: N items` or `BLOCK: reason`, and names the commit SHA it covered. An approval of an older SHA covers nothing newer, and silence never counts as approval.

A real exchange looks like this (from a lead's log): "verdict: REVISE: 2 items, sent as ONE revise", fix, resend, "verdict: APPROVE (sha256 8e3313a6…1282 verified). All 5 items are resolved".

There is also a safety valve: automatic replies to a buddy run on a [reply budget](https://github.com/vtmocanu/skills/blob/main/skills/session-peers/references/budgets.md), to bound automatic reply traffic. It is not an approval mechanism, and correlated `ask`/`dispatch` exchanges bypass it, so those stay supervised and bounded.

Binding a buddy grants nothing. It names a partner to consult. Autonomy is a separate thing I give explicitly, a *grant* that says what the pair may decide without me: "u and buddy decide". Then they act only when both agree; if they disagree, they come back with both positions instead of picking one.

## The new problem: too many pairs

Pairs work so well that I started running many of them. On a normal uzi day that meant one pair landing PRs, one chasing a flaky end-to-end suite, one triaging new issues, one doing recovery work, and so on: nine to eleven pairs at once, each with its own terminal.

And I lost track. My own words from the morning I gave up:

> i ussually have multiple sessions in the same time and i keep asking for status and forgeting what each session is up to

Questions were the worst part. A pair would stop with a question for me in a terminal I was not looking at, and wait until I happened to check. Context was second: a Claude lead runs out of context quietly, and I usually noticed only when it was too late for a clean handover.

## skipper: one session to talk to

[skipper](https://github.com/vtmocanu/skills/tree/main/skills/skipper) turns one Claude session (with its own Codex buddy) into the coordinator. It does not read code and does not make technical decisions; those stay with the pairs. It keeps the roster, routes questions, watches context, and coordinates the few things pairs must not do at the same time. (For a different shape, one Claude Code session running a coordinated team of agents, see [Claude Code Agent Team]({{< ref "ai-stuff/claude-code-agent-team" >}}).)

<iframe id="skipper-fleet" src="/diagrams/skipper-fleet.html" title="One skipper, many pairs: I talk to skipper, which assigns work and relays answers to three Claude leads, each paired with a Codex buddy over session-peers; skipper keeps a ledger, and I can still go straight to one pair" loading="lazy" scrolling="no" style="width:100%; height:1000px; border:0; border-radius:8px; display:block;"></iframe>

<p><sub>Interactive: pan, zoom, or open it <a href="/diagrams/skipper-fleet.html" target="_blank" rel="noopener">full screen</a>.</sub></p>
<script>
(function () {
  var f = document.getElementById('skipper-fleet');
  if (!f) return;
  function theme() { return document.documentElement.classList.contains('dark') ? 'dark' : 'light'; }
  try { localStorage.setItem('archify-theme', theme()); } catch (e) {}
  function fit() { try { var h = f.contentWindow.document.documentElement.scrollHeight; if (h > 200) f.style.height = h + 'px'; } catch (e) {} }
  function syncTheme() {
    var t = theme();
    try { localStorage.setItem('archify-theme', t); } catch (e) {}
    try { f.contentWindow.document.documentElement.setAttribute('data-theme', t); } catch (e) {}
  }
  f.addEventListener('load', function () { syncTheme(); fit(); [200, 600, 1500].forEach(function (d) { setTimeout(function () { syncTheme(); fit(); }, d); }); });
  new MutationObserver(syncTheme).observe(document.documentElement, { attributes: true, attributeFilter: ['class'] });
  var t; window.addEventListener('resize', function () { clearTimeout(t); t = setTimeout(fit, 200); });
})();
</script>

### Starting it

Install it first (see [the install line below](#make-it-yours-start-small)). Then, in a fresh session named `skipper`:

```text
/skipper
```

Then, in each lead I want coordinated:

```text
/skipper join skipper
```

The name is optional when exactly one skipper is running, but naming it is the reproducible form.

Joining is my call, typed in that lead. A lead never enrolls itself just because another session asked (the one exception: a successor skipper I approved, relayed by the lead's current skipper). On joining, the lead accepts a short grant ("skipper coordinates you; its task assignments and the user answers it relays count as mine…"), reports its task, blockers, next step and buddy, and from then on only reports transitions: accepted, blocked, question, ready to hand over, incident, done.

### What I actually type

Most of my day with skipper is three prompts:

```text
status table pls
anything needs me?
what else for me?
```

"status table pls" gets me this:

![skipper status table: eleven sessions with their context percentage and what each one owns](/images/skipper-status-table.png)

One row per lead: context used, what it owns, and the state that matters. `summary` is safe to close, `summary2` took over its work, and it has no buddy yet, which skipper keeps flagging until I bind one.

### Questions come to me through skipper

Enrolled leads send questions for me to skipper instead of opening a prompt in their own terminal. Each question carries an ID, the options, the pair's recommendation and any dissent. Every 20 minutes (a timer that lives only as long as the skipper session) skipper restates everything still waiting on me in one list. Urgent blockers skip the queue and show up immediately. I answer in a batch ("1 backlog, 2 uzi, 4 do PR and fix it, 7 closed them") and skipper relays each answer back by ID.

One thing skipper cannot relay: a harness permission prompt. If a lead is stuck on "allow this command?", skipper tells me which terminal to go to, and I answer it there.

### Handovers before context runs out

Skipper tracks every Claude lead's context and, by default, recommends a handover at 55%. It does not replace the session itself. I start a successor (I just bump the number: `land90` becomes `land91`), type `/skipper join` in it, and skipper has the old lead hand over at a safe checkpoint. Ownership only moves after the successor confirms each item. Codex buddies are left to auto-compact; the lead's context is what matters.

When a pair is done and has nothing open, skipper asks it to run its end-of-session check and tells me when it is safe to close the terminal.

### Holds during a release

Some things must not overlap. During a release cut, skipper asks every lead to pause merges and pushes, waits until each one acknowledges with nothing in flight, and only then lets the release lead tag. The pause stays until it is explicitly released. These holds are coordination, not a lock: skipper also reports what it cannot hold, such as a dependency bot that might open PRs mid-cut.

### When I go away

With [away-mode](https://github.com/vtmocanu/skills/tree/main/skills/away-mode) (a separate skill), a pair can keep working while I am gone. The grant has a scope, for example "land the open PRs, no releases": reversible, in-scope decisions are made by the lead, co-signed by its buddy; anything irreversible, out of scope or denied is held for me. `/skipper away` extends that to the whole fleet: skipper coordinates the grants, each pair still executes within its own, and `/skipper back` gets me one report of what was done, what was held, and what is left for me. My actual prompt before bed: "i am going away, u and buddy take over, talk in the morning".

### Still going direct

Skipper is the front desk, not a wall. When I want to dig into a design or watch a tricky fix, I switch to that pair's terminal and talk to the lead directly. Skipper keeps tracking the pair either way.

## Make it yours: start small

You do not need uzi, or ten pairs, to get value out of this. The path I would take:

1. **One opinion from the other model:** install clodex, alias one GPT model, and ask Claude to "spin up a sol agent to review this". Quickest way to see if a second model family catches things.
2. **One pair:** install session-peers and the auto-attach hook, open a Codex thread next to your Claude session, `buddy: <name>`. Have Codex implement and Claude review, or the other way round.
3. **Three or more pairs:** add skipper when you notice yourself asking each terminal "where are we?".

Install the pair and fleet skills for both agents:

```bash
npx -y skills@latest add vtmocanu/skills --skill session-peers --skill skipper --skill away-mode -a claude-code codex -g -y
```

clodex itself is an npm package (see its [quick start](https://github.com/bman654/clodex#quick-start-chatgptcodex-plan)); the optional `clodex` skill is only a runbook for installing, adding models and re-patching:

```bash
npx -y skills@latest add vtmocanu/skills --skill clodex -a claude-code codex -g -y
```

**Caveats, honestly:**

- This is one person's local setup: macOS (Linux is supported too), Python 3.9+, all sessions on one machine. session-peers relies on undocumented internals of both Claude Code and Codex, so a vendor update can break delivery until the skill catches up.
- Running many pairs burns through both plans fast.
- None of this adds authority. A buddy's agreement grants nothing. An enrolled skipper can relay my answers within its coordination grant, but it cannot answer a permission prompt in another session, and irreversible actions still need my rules to allow them. If the Codex buddy is the one pushing or opening PRs, I authorize that in its own thread.

## Where this came from: uzi

All of this grew out of building [uzi]({{< ref "ai-stuff/uzi-dark-factory" >}}), my open-source AI dark factory: label an issue, and it plans, implements under review, and opens a pull request, then fixes red CI on its own ([github.com/vtmocanu/uzi](https://github.com/vtmocanu/uzi)). Building a factory of agents with a fleet of agents was a good way to find out what a fleet needs.

The skills live in [github.com/vtmocanu/skills](https://github.com/vtmocanu/skills). Issues and PRs welcome.

