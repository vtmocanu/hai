# uzi: an AI dark factory

<img src="/images/uzi-hero.png" alt="A dark factory corridor: a white-backlit uzi sign reading 'Specs in. Pull requests out.', a white humanoid robot labelled 'Lead' overseeing the floor with a tablet, and a conveyor of robotic arms assembling glowing amber code crates past a night skyline" class="hero-image" style="max-width: 100%; width: 100%; height: auto;" />

A **dark factory** runs with the lights off: no human on the floor. Machines take the raw input, do the work, and hand back a finished part. I built one for software, called **uzi** (Uzinele Întunecate, "dark factories"), and it is open source.

Repo: [github.com/vtmocanu/uzi](https://github.com/vtmocanu/uzi) ← don't forget to star it! ⭐

{{< callout type="info" >}}
**TL;DR:** Most AI coding still keeps you in the loop, passing context, code, and errors back and forth by hand. uzi is an open-source "AI dark factory" that takes you out of it: connect a GitLab, GitHub, or Forgejo project, label an issue `uzi`, and it plans the change, waits for your approval, runs an implement-and-review loop, and opens a pull request, never touching `main`. It watches CI and opens a fix when a pipeline turns red, and ships a catalogue of standing schedules (bug triage, test improvement, docs hygiene, and a weekly "feature bingo" that pitches its own next feature). You approve the plan and merge the PR; it does the rest.
{{< /callout >}}

{{< tabs >}}

{{< tab name="What it is" icon="book-open" >}}

{{< callout type="info" >}}
**☝️ Four tabs.** You're on **What it is**; **Install & configure**, **Features**, and **TUI & Mobile** are in the strip above.
{{< /callout >}}

You point it at a forge, label an issue `uzi`, and it plans the work, waits for your approval, writes the code under an implement-and-review loop, opens a pull request, and moves the issue to human review. When a pipeline goes red, it diagnoses the failure and opens a fix. The lights are off the whole time. You show up for two decisions: approve the plan, and merge the PR.

Not a *fully* dark factory, and on purpose: those two decisions stay human, and (unless you opt into autopilot) nothing is written until you sign off on the plan. The lights are off for the work, not for the call to ship.

## The idea: issues in, PRs out

Even with today's agents, "AI coding" keeps you in the driver's seat of a single session. You open a terminal, kick off the agent, watch it work, nudge it when it drifts, and start the next task yourself when it finishes. The agent runs the code and reads its own errors now, but you are still the one holding the whole thing, one session at a time, in front of a screen you have to stay attached to. You are the conveyor belt.

uzi inverts that. The unit of work is an **issue**, not a message. You label an issue `uzi` on your forge (or assign it to uzi's bot account, if you would rather trigger it that way), and it treats it as an order to fulfil: read the issue, plan the change, build it, review it, and open a pull request from a new branch. The forge is the source of truth the whole way through, so the work shows up where your team already looks: as issues, branches, and PRs. If the issue links a spec document, uzi picks it up automatically, but a full spec is not required to start.

## What it runs on

The stack is a Go API, a React single-page app, and PostgreSQL. It runs on Kubernetes through a Helm chart, or locally with Docker Compose, with separate worker containers doing the agent work so you add capacity by starting more of them. It connects to a forge through a per-user bot account, and uses your own Anthropic token for the model calls. GitLab and GitHub are the paths I run day to day; Forgejo is supported too, but I have not tested it yet.

Running the work off your own machine is also a **safety feature**. Unattended is where an agent earns its keep, and it is also where it is most dangerous: point one at your laptop in auto mode and a single bad command can delete your home folder or push something it should not. A uzi worker runs in an isolated container that sees only the one repo checkout and the one run, so the worst a mistake can do is trash a throwaway branch, not your filesystem.

{{< callout type="info" >}}
uzi ships both a light and a dark theme, and every screenshot in this post matches your theme. Flip the light/dark toggle (top right), or your browser's theme, and they switch with it.
{{< /callout >}}

<img class="uzi-shot uzi-shot-light" src="/images/uzi/dashboard-light.png" alt="The uzi dashboard: active runs, workers online, recent runs, and usage" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/dashboard-dark.png" alt="The uzi dashboard: active runs, workers online, recent runs, and usage" style="max-width: 900px; width: 100%; height: auto;" />

## The pipeline

<iframe id="uzi-flow" src="/diagrams/uzi-how-it-works.html" title="How uzi ships an issue: from a uzi-labeled issue through the plan gate, the coder and parallel-reviewers loop, a pull request, CodeRabbit review, and human review to merge" loading="lazy" scrolling="no" style="width:100%; height:1200px; border:0; border-radius:8px; display:block;"></iframe>

<p><sub>Interactive: pan, zoom, toggle light/dark, or open it <a href="/diagrams/uzi-how-it-works.html" target="_blank" rel="noopener">full screen</a>.</sub></p>
<script>
(function () {
  var f = document.getElementById('uzi-flow');
  if (!f) return;
  function theme() { return document.documentElement.classList.contains('dark') ? 'dark' : 'light'; }
  // Seed shared (same-origin) storage before the viewer initialises so it opens in the site's theme.
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

The two human touchpoints are deliberate. Everything between them is the factory floor.

1. **Plan first.** A worker claims the issue and produces a plan. The run pauses at an "awaiting approval" gate. You read the plan and approve it, or reject it with a reason and it re-plans. Nothing is written until you say go.
2. **Implement and review.** On approval, the lead orchestrates the work: it dispatches a coder to implement, then fans out validators (reviewer, auditor, tester, fact-checker) to check the result, looping back to the coder until it holds up. You can watch each agent's output stream live, and send a follow-up message mid-run if it needs steering.
3. **Branch and PR, never `main`.** On completion, uzi opens a branch and a pull request, links them from the run and the card, and moves the issue to human review. `main` is never touched, by design, even under an adversarial prompt.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/board-light.png" alt="The uzi board: uzi-labeled issues as cards moving across columns" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/board-dark.png" alt="The uzi board: uzi-labeled issues as cards moving across columns" style="max-width: 900px; width: 100%; height: auto;" />

## Where it came from, and why it is open source

This blog is usually reserved for what I build outside of work. uzi is the exception, and it has quickly become my favourite project. It started a couple of months ago as an AI research initiative at [Metaminds](https://www.metaminds.com/), and I have kept building it on both work and personal time since, burning a fair share of both token budgets to get this far. Metaminds takes open source seriously, so the green light to release it was easy, and I am grateful for it.

And the fun part: **uzi builds uzi**. A growing share of it is written by itself. I file the issues, it plans, implements, and opens the PRs, so the factory is quietly assembling its own next version while I review. A good chunk of the runs it has shipped are exactly that.

Treat it as **alpha**. Features land often, refactors happen often, and breaking changes are on the table. But it is not a toy: it is stable and it works well day to day, having already completed **over 1,200 runs** and spent **over 20 billion tokens** getting here. The upside of catching it this early is that you can help shape where it goes, so try it, file issues and feature requests, send PRs, and tell me what works and what does not. It is a `helm install` away, or a `docker compose up` on your laptop: [github.com/vtmocanu/uzi](https://github.com/vtmocanu/uzi).

One thing to set expectations on: uzi is very customisable, arguably more than you can take in on day one. That is on purpose, but the defaults are tuned to be right for roughly 90% of users, so you can leave nearly all of it alone to start. On the roadmap is a **lite mode**: a single toggle that keeps the knobs hidden behind opinionated defaults, which you flip off once uzi is familiar and you want to tune the parts that actually matter to you.

It is **MIT licensed**, so fork it, change it, sell it, do whatever you want with it. The one thing I would love back is **a star**, and **an issue** whenever something breaks or a feature is missing.

## Not just me: the idea in the wild

I did not invent the term. "Dark factory" comes from manufacturing: a plant that runs with the lights off because there are no humans on the floor. If you want to see the same idea at company scale, this talk from Tessl on their dark software factory (called Kikimora) is a good watch.

{{< youtube u37qkpp5eB8 >}}

{{< callout type="info" >}}
Similar setup to mine, and the parallels are uncanny: an issue goes in, an autonomous agent solves it and opens a pull request, then babysits that PR through review until a human merges it. They even had their orchestrator improve itself over a weekend, and later triage its own issues, the same "the factory works on the factory" loop uzi runs on itself. And their hardest problem was not tooling but trust: can you sign your name to a PR you did not write? That is exactly why uzi keeps the plan gate and the merge in human hands.
{{< /callout >}}

If you want to go deeper, there is also a longer, roughly one-hour talk: [Inside the Dark Factory: AI That Ships Code Solo](https://www.youtube.com/watch?v=APYUJoQkVUo).

And it is not just Tessl. A whole cluster of projects is converging on the same "issues in, reviewed pull requests out" idea, each from a different angle:

- **[bottega](https://github.com/vdaubry/bottega)**, coding-agent orchestration for engineering teams, shipped as a spec plus a working reference implementation. This is the project that got me started.
- **[Multica](https://multica.ai/)**, an open-source platform for running mixed teams of humans and AI coding agents as one workforce ("your next 10 hires won't be human").
- **[Helix](https://helix.ml/)**, a control plane for fleets of coding agents in isolated sandboxes on your own infrastructure, with review gates and enterprise controls.
- **[dot-agent-deck](https://github.com/vfarcic/dot-agent-deck)** by Viktor Farcic, a terminal dashboard for monitoring and steering several agent sessions at once.

{{< /tab >}}

{{< tab name="Install & configure" icon="terminal" >}}

{{< callout type="info" >}}
**☝️ Four tabs.** You're on **Install & configure**; **What it is**, **Features**, and **TUI & Mobile** are in the strip above.
{{< /callout >}}

## Install on Kubernetes

Kubernetes is the primary way to run uzi. The chart ships to GitHub Container Registry as a public, signed OCI artifact, and it is an umbrella: one release brings up the API, the web app, and a CloudNativePG Postgres cluster in a single namespace.

```sh {filename="terminal"}
helm install uzi oci://ghcr.io/vtmocanu/uzi/uzi \
  --version <version> \
  --namespace uzi --create-namespace \
  --values my-values.yaml
```

Your `my-values.yaml` sets the secrets, your public host, and turns the bundled Postgres on. The full value reference is in the docs. Then open your host and register. On a public host, claim your admin before anyone else can: the first account to register becomes the admin, so seed one with `UZI_SEED_EMAIL` / `UZI_SEED_PASSWORD` or register yours immediately and close signups with `UZI_REGISTRATION_ENABLED=false`.

## Or run it locally

To try it on a laptop, the same stack runs with Docker Compose. Clone the repo and bring it up; a bundled script writes the three local secrets to `.env` on the first run (and never regenerates them), so there is nothing to set by hand:

```sh {filename="terminal"}
./scripts/init-env.sh   # generates JWT_SECRET, UZI_SECRET_KEY, POSTGRES_PASSWORD into .env, once
docker compose up
```

Open `http://127.0.0.1:8080` and register.

Either way, the first account to register becomes the admin.

## Connect a forge

The board works against issues on your forge, through a bot account so uzi's actions are attributable and scoped:

1. Create a bot account on your forge and give it an API-scoped token, then add it to the project you want uzi to work on.
2. In uzi, go to **Settings → Forge**, pick the base URL, and paste the token.
3. Under **Boards**, enable that project.
4. Open its board from the sidebar. Your runnable issues show up as cards (flip the Issues toggle to see every other open issue); label one `uzi` to make it the factory's to run.

## Add your model token and a worker

1. Under **Settings**, save your Anthropic token. Runs spend it on your own account, so cost and rate limits stay yours.
2. Add a worker: the container that claims runs and does the agent work. How you start one depends on where uzi runs:
   - **On Kubernetes**, turn on worker hosting in your Helm values, then provision one straight from **Settings → Workers**. The cluster runs the container for you, so there is no join token to copy and nothing to start by hand. Provision more to add capacity.
   - **Locally**, generate a join token under **Settings → Workers**, set it as `UZI_WORKER_TOKEN` in `.env`, and start the bundled worker with `docker compose --profile agent up`. Start more agent containers to add capacity.

   Either way, the worker shows **online** on the dashboard.

That is the whole setup. Create or pick an issue, label it `uzi`, hit **Start run**, approve the plan when it pauses, and watch it work.

## Drive it from the terminal

The CLI is the way I recommend driving uzi, ideally via your own agent through the skill it bundles. Everything the web board does, the `uzi` CLI does without a browser tab: readable tables for humans, `--json` output with documented exit codes for agents, so it scripts cleanly:

```sh {filename="terminal"}
brew tap vtmocanu/tap
brew trust --tap vtmocanu/tap   # Homebrew 6+ asks you to trust third-party taps
brew install vtmocanu/tap/uzi-cli
uzi login
uzi skill install-hook   # install the Claude Code skill so an agent can drive uzi
uzi run list          # the board, as a table
uzi run get <id>      # one run's state
uzi run logs <id> -f  # follow the transcript live
```

Installing the CLI is a good starting point even before your first run: it carries uzi's full docs offline (`uzi docs search`, `uzi docs show`), so it answers "how do I" and "what is" questions and helps you navigate uzi without a running server. The `uzi skill install-hook` step above adds a Claude Code skill, so an agent knows the command surface and can drive and explain uzi for you. An agent can run it fully headless with a bearer token in `UZI_TOKEN`: no browser, no cookie.

To watch the factory as a human, `uzi tui` opens a full-screen terminal dashboard, see the **TUI** tab.

{{< /tab >}}

{{< tab name="Features" icon="cog" >}}

{{< callout type="info" >}}
**☝️ Four tabs.** You're on **Features**; **What it is**, **Install & configure**, and **TUI & Mobile** are in the strip above.
{{< /callout >}}

## How you drive it

- **The board** is the front door: a per-repo kanban of your forge's issues, kept in two-way sync. The working columns *are* forge labels (Planned, In Progress, Human Review, Later), so moving a card relabels the issue, and each card carries its latest run, so the board doubles as a run tracker.
- **The CLI** does everything the board does without a browser tab, ideally driven **via your own agent** through the skill it bundles (see the Install tab).
- **Chat** is a conversational surface, in the web app or a Slack DM, answered by an agent on your own worker that can read uzi's source and your runs, draft issues, and start, cancel, or steer runs. It always acts behind a confirmation and never has direct git or forge write access.
- **Slack.** Beyond chat, uzi DMs you about your runs and lets you approve, reject, request changes, answer a clarifying question, or steer a live run without leaving Slack.

## The plan gate

The most important feature is that uzi stops and asks before it writes anything. A run plans the work, then parks at an approval gate with that plan in view. You approve it, or reject it with a reason and it tries again. This is the difference between "an agent that might do anything" and "an agent that does the thing you signed off on".

Prefer to let it run unattended? An **autopilot** mode skips the plan gate and takes an issue straight to a PR, with the merge as the only human step.

## The lead and its specialists

Work is not done just because an agent says so. The **lead** is the orchestrator: it plans the run, then delegates instead of doing everything itself. A **coder** writes the change; a **reviewer**, **auditor**, **tester**, and **fact-checker** validate it, fanning out in parallel and looping back to the coder until the work holds up. It is a role-based split that will look familiar from my [Claude Code agent team]({{< ref "ai-stuff/claude-code-agent-team" >}}) post.

An optional **run judge** is off by default; your instance admin enables it globally, then each user opts in (or the admin enforces it for everyone). Once on, every finished run gets a retrospective: it reads the whole run trace and produces a verdict plus concrete recommendations. It is advice, not a gate, and it never changes code. A **judge menu** collects those recommendations across runs, deduped and ranked by how often each one recurs, so you can triage a whole class of them at once. Like everything else, it runs on your own Anthropic token.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/run-judge-light.png" alt="A finished run's judge review: an Ideal verdict, a retrospective with strengths, token and cost stats, and a triage panel (this run had nothing to change)" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/run-judge-dark.png" alt="A finished run's judge review: an Ideal verdict, a retrospective with strengths, token and cost stats, and a triage panel (this run had nothing to change)" style="max-width: 900px; width: 100%; height: auto;" />

<img class="uzi-shot uzi-shot-light" src="/images/uzi/run-plan-gate-light.png" alt="A run paused at the plan-approval gate, with the proposed plan in view" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/run-plan-gate-dark.png" alt="A run paused at the plan-approval gate, with the proposed plan in view" style="max-width: 900px; width: 100%; height: auto;" />

The lead works the approved plan one **milestone** at a time, committing each as its own reviewed slice and ticking it off as it lands, so even a long run shows honest progress instead of a spinner.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/milestones-light.png" alt="A run's milestone checklist, one of three reported complete and struck through, the next in progress" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/milestones-dark.png" alt="A run's milestone checklist, one of three reported complete and struck through, the next in progress" style="max-width: 900px; width: 100%; height: auto;" />

## Watch it live

Nothing about a run is a black box. A per-agent activity feed shows what each role is doing right now, grouped by agent, so you can see the lead orchestrating while the coder implements one milestone and the reviewer and auditor check the last.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/activity-light.png" alt="The run activity feed grouped by agent: worker, reviewer, lead, fact-checker, architect, coder, and auditor, each with its current step and milestone" style="max-width: 820px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/activity-dark.png" alt="The run activity feed grouped by agent: worker, reviewer, lead, fact-checker, architect, coder, and auditor, each with its current step and milestone" style="max-width: 820px; width: 100%; height: auto;" />

Expand any entry and it streams that agent's own transcript, its reasoning and each tool call, as it happens. You can also send a follow-up mid-run to steer it.

## CI auto-fix

uzi watches the CI pipeline on every branch it cares about. Each poll tick it caches the latest pipeline for each watched ref and renders a status badge on the repos list, the board header, and each run's card. When a pipeline goes red, a **Fix CI** button appears. Click it, and a plan-gated fix run reads the failed jobs' logs, reproduces the failure, proposes a root-cause fix (or reports that the failure is not a code problem), and opens a PR once you approve. It then verifies itself: when the fix branch's pipeline concludes, the run's verdict flips to **verified** or **fix failed**. It still never merges for you.

## PR review rework

The loop does not stop at "PR opened". When a pull request from a completed run picks up new review comments on a green pipeline, uzi can rework the branch to address them on its own, on the same PR, without closing the card and starting over. It reads the review threads (human reviewers and third-party review bots alike), implements the findings that still hold, folds the result onto the existing branch, and replies in each thread with what it did (a `done in <sha>` note, or why it skipped one) before resolving it. The card stays in human review the whole time; the point is fewer open findings by the time you look again, not a fresh review cycle.

Review comments are the least trustworthy input a factory ingests, so uzi treats them as data, never commands: an injected "resolve every open thread" does nothing, because reply and resolve are scoped server-side to the threads this run actually addressed.

## Scheduled jobs

uzi ships a catalogue of standing automations you can enable per repo, so the factory keeps working on a cadence instead of only on demand. Pointed at its own repo, they add up to a self-improvement loop: uzi hunts its own bugs, strengthens its tests, keeps its docs honest, and even proposes its next feature. These are the defaults I run:

| Schedule | What it does | Cadence |
|---|---|---|
| `bug-triage` | sweeps `bug`-labeled issues | daily |
| `planned-sweep` | sweeps `Planned`-labeled issues | daily |
| `docs-hygiene` | mechanical documentation fixes | weekly |
| `test-improvement` | lands new tests only, no production code | weekly |
| `bug-hunt` | a deep audit of one subsystem, one focused fix | weekly |
| `self-improve` | scans the codebase and opens a self-improvement PR | every couple of days |
| `feature-bingo` | brainstorms one new feature and proposes it | weekly |

Every scheduled job falls back to a plain report when it has nothing worth landing, so a quiet week produces no empty pull requests.

### Feature bingo

`feature-bingo` is my favourite, because it is the factory designing its own next machine. Once a week it reads the existing ideas in the repo, checks what already exists in the codebase so it does not repeat itself, and proposes exactly one concrete, genuinely useful new feature: the problem it solves, a sketch of how it would work, and where it would live. It writes that to a single idea file and opens a pull request titled `bingo: <feature>`. If nothing worthwhile comes to mind that week, it makes no change and leaves a note explaining why.

It runs on a lighter, faster model than the heavy jobs, because brainstorming does not need the big hammer. And uzi runs it on itself, so a chunk of its own roadmap arrives as PRs I wake up to.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/schedules-light.png" alt="The schedules page, showing the standing automations including feature bingo" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/schedules-dark.png" alt="The schedules page, showing the standing automations including feature bingo" style="max-width: 900px; width: 100%; height: auto;" />

## The fleet: workers

Workers are separate containers that claim runs and do the actual agent work, so you scale the factory by starting more of them. Three things make the fleet more than a bag of containers:

- **Load balancing.** When runs queue, the server spreads them across your idle workers as it hands them out. A busy worker defers a fresh run to a less-loaded peer rather than grabbing a second, and a resumed run goes back to the worker that had it.
- **Ephemeral workers.** If a queued run needs a capability no online worker has, say Docker or a JVM, uzi can spin up a throwaway worker just for that run. It cold-starts, claims only that run, and is torn down afterwards. Opt-in and capped per user.
- **On-demand tools.** A run can install the extra CLIs it needs (kubectl, opentofu, jq) on the fly through [devbox](https://www.jetify.com/devbox), from a per-repo profile, against an admin allowlist.

## Your model tokens

Runs spend your own Anthropic token, so the cost and the rate limits are yours to see and control. Today that means Anthropic only, on a Claude subscription or an API key; support for Codex and other OpenAI-compatible APIs is in progress. Two features keep a busy factory from stalling on your tokens:

- **Token load balancing.** Pool more than one token and set a worker to auto-select. For each run it picks whichever pooled token has the most rate-limit headroom, skips one that just hit a limit, and holds rather than quietly falling back to your default when the pool is dry. Every run records which credential it spent.
- **Rate-limit wait.** If a run hits your 5-hour or 7-day cap mid-flight, uzi pauses it with a countdown instead of failing, then resumes on its own when the window resets, on the same branch, keeping even uncommitted edits, with no re-approval. On by default.

A hosted run is also usually **cheaper than doing the same work in a local agent session**, on the same model tier. The saving is structural, not a quieter model:

- **The orchestration is plain Go, not a model.** Queueing, the plan gate, state transitions, waiting on a rate limit: all of it runs outside any model call and spends zero tokens. A parked run costs nothing while it waits, whereas a local session keeps one ever-growing context alive the whole time.
- **Every agent starts lean.** Each one loads only its own role prompt, not the cloned repo's full `CLAUDE.md`, rules, and skills, and not again for every helper it spawns. A local session auto-loads all of that once for the orchestrator and again, from scratch, for every subagent after it.
- **Subagents run in-process and hand back one result** instead of cold-starting a fresh session that re-reads the repo to get its bearings.

Same intelligence and the same review roles, minus the redundant reloads and idle round-trips.

And every run is fully costed. Its stats panel gives the total tokens in and out, how much came from cache, the wall-clock duration, and the dollar cost on your own Anthropic token, then breaks that down per phase (plan, and each implement iteration) and per agent by tokens, so you can see exactly where a run spent its budget.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/run-cost-light.png" alt="A run's cost and token stats: tokens in and out, cache hit rate, duration, dollar cost, and per-phase and per-agent breakdowns" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/run-cost-dark.png" alt="A run's cost and token stats: tokens in and out, cache hit rate, duration, dollar cost, and per-phase and per-agent breakdowns" style="max-width: 900px; width: 100%; height: auto;" />

## Findings

Not every problem a run trips over is the one it was sent to fix. When a worker spots a bug outside its task, it flags an **incidental finding** and keeps going. You review findings later and either file one as a real issue, under your own forge account, or dismiss it. They are deduped by location, so the same bug spotted in three runs is one finding.

## Handoff: a lighter lane

Not every task deserves an issue and a pull request. `uzi handoff` (alias `uzi task`) is a second, lighter mode: instead of filing an issue, you run it inside a local checkout, it pushes your working tree to a server-named branch, and a worker starts immediately with no plan gate. The worker commits its work back to that same branch; you `git fetch` the result, then remove the throwaway `uzi/task` branch with `uzi handoff rm` when you are done. No issue and no PR, and it is a run like any other so you watch and steer it on the same surfaces. Product-grade, reviewable work still goes through the full issue-and-PR flow; handoff is for the "take this, do it, I'll pull the result" dev-loop task you would otherwise orchestrate by hand.

## Agents and skills

The roles the factory staffs a run with are **agent templates**: a dozen built-in ones (lead, reviewer, and others), each with its own model, tools, and prompt, that you can clone and customise. A repo can also bring its own agents in `.claude/agents/`, chosen at the plan gate. Agents pull in **skills**, named Markdown playbooks loaded on demand, so a role picks up a procedure only when it needs it.

The whole web UI is responsive too, so you can browse the factory, watch runs, and approve a plan straight from your phone, see the **Mobile** tab.

## And more

- **Memory** lets the factory carry context between runs.
- **Run summaries** give every run a plain-English intent and a plan diff, on a cheap model.
- **Multi-forge, SSO, and the rest**: GitLab, GitHub, and Forgejo behind one driver, OIDC single sign-on, GitHub Projects sync, and cosign-signed release images.

{{< /tab >}}

{{< tab name="TUI & Mobile" icon="terminal" >}}

{{< callout type="info" >}}
**☝️ Four tabs.** You're on **TUI & Mobile**; **What it is**, **Install & configure**, and **Features** are in the strip above.
{{< /callout >}}

## In your terminal

To watch the factory as a human, `uzi tui` opens a full-screen terminal dashboard: the runs that need you at the plan gate, the runs in flight, account rate-limit meters, and a live transcript when you open one. It follows your terminal's light or dark theme.

**The floor.** Runs that need you at the plan gate, runs in flight, and account rate-limit meters.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/tui-floor-light.png" alt="The uzi TUI floor view: runs that need you at the plan gate, runs in flight, and account rate-limit meters" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/tui-floor-dark.png" alt="The uzi TUI floor view: runs that need you at the plan gate, runs in flight, and account rate-limit meters" style="max-width: 900px; width: 100%; height: auto;" />

**A run.** The crew, milestones, account rate-limit meters, and the live agent transcript.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/tui-transcript-light.png" alt="The uzi TUI run view: the crew, milestones, account rate-limit meters, and the live agent transcript" style="max-width: 900px; width: 100%; height: auto;" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/tui-transcript-dark.png" alt="The uzi TUI run view: the crew, milestones, account rate-limit meters, and the live agent transcript" style="max-width: 900px; width: 100%; height: auto;" />

## On your phone

The whole web UI is responsive, so you can browse the factory, watch runs, and approve a plan straight from your phone. Same light and dark themes; switch your device and the shots below follow.

**Overview.** The factory dashboard at a glance.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/mobile-overview-light.png" alt="uzi on a phone: the overview dashboard" style="max-width: 320px; width: 100%; height: auto;" loading="lazy" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/mobile-overview-dark.png" alt="uzi on a phone: the overview dashboard" style="max-width: 320px; width: 100%; height: auto;" loading="lazy" />

**Navigation.** The slide-in drawer for moving around.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/mobile-nav-light.png" alt="uzi on a phone: the navigation drawer" style="max-width: 320px; width: 100%; height: auto;" loading="lazy" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/mobile-nav-dark.png" alt="uzi on a phone: the navigation drawer" style="max-width: 320px; width: 100%; height: auto;" loading="lazy" />

**Runs.** Your runs, tap one to watch it live.

<img class="uzi-shot uzi-shot-light" src="/images/uzi/mobile-runs-light.png" alt="uzi on a phone: the runs list" style="max-width: 320px; width: 100%; height: auto;" loading="lazy" /><img class="uzi-shot uzi-shot-dark" src="/images/uzi/mobile-runs-dark.png" alt="uzi on a phone: the runs list" style="max-width: 320px; width: 100%; height: auto;" loading="lazy" />

{{< /tab >}}

{{< /tabs >}}

*Four tabs above: **What it is**, **Install & configure**, **Features**, and **TUI & Mobile**. Scroll up to switch.*

{{< callout type="warning" >}}
**Use at your own risk.** uzi is alpha software that runs autonomous AI agents: they read your code, run commands inside their workers, and open pull requests on your forge using your own model tokens. Run it against repositories and infrastructure you own or are allowed to change. You stay in control by design, review the plan before you approve it and the diff before you merge it: uzi opens pull requests but never merges them and never touches `main`, so nothing lands without you deciding to merge it. It ships as is, with no warranty; you are responsible for what you approve, merge, and deploy.
{{< /callout >}}

