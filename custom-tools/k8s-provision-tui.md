# k8s-provision-tui: the DR automation I didn't know I was building

<img src="/images/k8s-provision-tui.png" alt="A relaxed operator at a dim command-center console watching a TUI dashboard with coloured status glyphs, a blue cluster rack running steadily on the left, a green cluster rack being rebuilt by a small cartoon robot on the right, and a dusty 'RUNBOOK' book discarded on a side table" style="max-width: 700px; width: 100%; height: auto;" />

I already run my Kubernetes clusters in a blue/green pattern: new color up, data migrated, old color torn down, every piece of it reproducible from git. The [data-integrity problem](/migrations/s3bkp-to-volsync/) was solved. What I hadn't solved was the ceremony of actually doing the switch.

{{< callout type="info" >}}
**Skimming?** There are screenshots further down under [What the TUI actually does](#what-the-tui-actually-does) if you'd rather see the thing before reading the story.
{{< /callout >}}

## The procedure I kept postponing

The cutover lived in a wiki page. It was long. Really long. Dozens of steps, each one a shell command or a kubectl pipeline or a click in the Spectro Palette UI, with prerequisites that mattered, ordering that mattered, and invariants ("the live color must not be the target color", "CNPG hibernation must happen after Flux is suspended") that were easy to forget at step 17 when you're tired.

Every step was documented. Every step was correct. It just took hours, demanded the kind of attention where one misstep would blow away live backups, and ate a full afternoon every time. **So the procedure worked the way long procedures always work:** I read it, **dreaded it, and postponed** whatever triggered the need to run it. The longer I postponed, the more drift accumulated in the old cluster, the scarier the switch felt, the longer I postponed. You know the loop.

## From a bash script to a TUI

It started, as these things do, with a bash script. I wrapped the worst offenders first: a function for `tofu apply` in the right directory with the right `TF_COLOR`, another for the Ansible playbook against the fresh VMs, a third that checked backup freshness before the destructive phase. Then a menu using [gum](https://github.com/charmbracelet/gum) so I didn't have to remember flags. Then a second menu. Then state I needed to thread between menus and couldn't, because it was bash.

At that point the script was hundreds of lines of bash, the menus were nested three deep, and I was fighting the language every time I wanted to show a progress indicator or pass structured data between steps. I rewrote it in Python using [Textual](https://textual.textualize.io/) and the difference was immediate: what took 30 lines of `gum`-and-`read` gymnastics became a proper modal with focus handling and keyboard shortcuts.

A few weekends later I had a TUI with a live status dashboard, an accordion menu driving every phase of the procedure, and type-to-confirm guards on the destructive actions. The wiki page still exists; it's now the specification that the TUI implements, not something I follow by hand.

What I kept was the operator's judgment. The TUI does not run the whole procedure unattended. Every destructive step still confirms. Every mutation still shows the plan before it fires. But everything between those confirmations (listing VMs, polling the Palette API through rate limits, waiting for DNS to propagate) is done by the TUI, visibly, with a live status trail.

Reinstalling a cluster now fits in a Saturday afternoon. I don't dread it. I actually enjoy it, because watching the dashboard light up green row by row as the new cluster stands up is genuinely satisfying.

## The realization

Somewhere around the fourth or fifth iteration, I noticed something odd. The TUI wasn't really about reinstalls. Everything it did (bucket wipes, DNS flips with rollback, liveness guards against wiping the wrong cluster) was disaster recovery.

A planned blue/green reinstall is a controlled DR drill. The cluster is deliberately lost. The workloads are deliberately restored. The only difference between "I reinstalled green" and "green burned down in a fire" is whether the old cluster is still running during the restore. The actions I automate are the same. The safety invariants are the same. The backup dependencies are the same.

I was building a DR automation tool and calling it a reinstall helper.

That reframe changed how I think about the TUI. Every feature I'd added to make the cutover safer (liveness checks, type-to-confirm for irreversible steps, backup-age gates) wasn't just ergonomics. It was disaster recovery tooling. Using the TUI regularly for planned reinstalls, I'm continuously testing my actual DR path. The procedure I used in anger had already been rehearsed dozens of times.

## The runbook problem

There's a broader point hiding in this. A written runbook (a Confluence page, a wiki, a `RUNBOOK.md`) is a static description of a moving system. You read it top to bottom, you copy-paste the commands, and somewhere around step 14 you skip a line, or you paste a command with the wrong argument, or the step you're on assumes a precondition that silently isn't true anymore. That's how incidents happen during planned work.

A TUI with a live status dashboard flips that: the current state is always in front of you, the next safe action is the one you can see lit up, and the tool refuses to fire a step whose precondition isn't met. The runbook stops describing the system and becomes part of it.

## What the TUI actually does

It's a single-screen Textual app, split into three regions: a live status dashboard at the top, an accordion menu underneath, and a status bar at the bottom.

<img src="/images/k8s-provision-tui-start.png" alt="k8s-provision-tui at startup: live status dashboard listing 16 check rows grouped by phase, with colored status glyphs for each" style="max-width: 900px; width: 100%; height: auto;" />

Each dashboard row is one check: VM templates on the Proxmox nodes, registration token freshness in Palette, edge hosts registered, cluster profile deployed, Flux controllers running, backup freshness per system (VolSync, CNPG, K10, Velero), and so on. The glyphs (●/◐/○) track state in real time. A background worker re-runs the checks every few seconds, and `r` forces a refresh on demand. The rows for the current phase of the cutover are where the operator's eye naturally goes; the rest are a running check that the rest of the homelab isn't on fire while I'm mid-cutover.

The menu mirrors the phases of the wiki runbook. Expanding a phase reveals its actions, each one annotated with its own status glyph based on the live dashboard state:

<img src="/images/k8s-provision-tui-menu-expanded.png" alt="Accordion menu with Setup, Pre-cutover, Backups, Drain, and Cutover phases expanded, each action labeled with a live status glyph and a short summary" style="max-width: 900px; width: 100%; height: auto;" />

Triggering an action never fires the underlying command blindly. Every action first shows a preview modal with the exact plan: which VMs, which clusters, which files, and the command chain that will run. For mutating actions the default button is "No", so you have to deliberately say yes.

<img src="/images/k8s-provision-tui-run-ansible.png" alt="Preview modal for 'Run Ansible' showing the full command chain that will execute, with Yes/No buttons" style="max-width: 900px; width: 100%; height: auto;" />

Once confirmed, output streams into a dashboard pane in place of the checklist. This was the single best decision I made while building this: no `app.suspend()` to hand the terminal to ansible, no scrollback to hunt through later. The TUI stays on-screen, the subprocess output streams line by line into a log widget, and a "Press Enter to return" modal closes the action cleanly when it finishes.

<img src="/images/k8s-provision-tui-ansible-output.png" alt="Ansible playbook output streaming live into the dashboard CommandLog pane while the TUI menu stays visible around it" style="max-width: 900px; width: 100%; height: auto;" />

The same streaming pattern covers API-driven actions too. The most destructive ones get an extra gate on top of the Yes/No: the bucket-wipe action runs `mcli rm` against four per-color backup buckets, so before it fires the confirm modal makes me type the color name (`blue` or `green`) literally. The same pattern guards the Ceph pool reset, where I have to type the pool name. Once past that, the pane shows each command, each per-bucket result, and a final summary modal:

<img src="/images/k8s-provision-tui-bucket-clean.png" alt="Bucket cleanup action streaming output: per-bucket rm commands, their results, and a summary of objects wiped" style="max-width: 900px; width: 100%; height: auto;" />

## Where does this leave you?

If any of this resonates, do the thing. Open your DR plan (you have one, right?), point the AI agent of your choice at it, and tell it to build a TUI for it. Start small. Iterate. The first version won't be pretty; mine was hundreds of lines of bash and a nested gum menu. That's fine. The payoff isn't the TUI, it's that every time you use it you're rehearsing a restore you'd otherwise have done zero times.

My TUI is ~4,000 lines of Python, intentionally specific to my homelab's blue/green layout, and one of the most satisfying things I've built this year.

