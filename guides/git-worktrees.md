# Git Worktrees

<img src="/images/git-worktrees.png" alt="Git Worktrees" style="max-width: 700px; width: 100%; height: auto;" />

Most developers use one working directory per repo. When you switch branches, git overwrites the files in place. That works fine until you need two branches at the same time, or three, or you want AI to try a big refactor without touching your current work.

Git worktrees fix that. They let one repo have multiple working directories, each on its own branch, sharing the same history.

I use them heavily now, especially for AI-assisted work, where being able to throw away an experiment cheaply is the difference between safe exploration and a painful cleanup.

## What a worktree actually is

A worktree is a directory with a working copy of your repo, pinned to a specific branch, that shares the single underlying `.git` object database with the main clone.

- **Standard clone:** one `.git` directory, one working tree, one branch at a time.
- **Worktree setup:** one `.git` (or one bare repo), *many* working trees, one branch each, all in parallel.

Worktrees have been in Git since version 2.5 (2015). Not new, just underused.

## Why worktrees matter now

Two reasons specific to how I work today:

1. **AI can refactor aggressively.** Ten commits, half your files renamed, three new abstractions you didn't ask for, all in 30 minutes. If that's on `main`, untangling is painful. If it's on a worktree, you delete the directory and move on.
2. **Parallel AI sessions.** I often run two or three Claude Code sessions at once. They fight over the working directory if they share it. Worktrees let each session own its own checked-out copy.

Worktrees are also great for the classic "hotfix while mid-feature" interruption, but that was true in 2015 too.

## Basic usage

From inside your repo:

```bash
# Add a worktree for an existing branch
git worktree add ../hotfix hotfix-branch

# Add a worktree AND create a new branch
git worktree add -b feature/payments ../payments main

# List all worktrees for this repo
git worktree list

# Remove one when done
git worktree remove ../hotfix

# Clean up references to worktrees you deleted manually
git worktree prune
```

Git prevents the same branch from being checked out in two worktrees at once. This is by design: it would let you have two conflicting working copies of the same branch. Use a new branch (or the `-b` flag) instead.

## Two layouts you'll run into

### Regular clone (sibling worktrees)

What you get from `git clone`:

```treeview
~/projects/my-repo/       ← main working tree (.git is here)
├── .git/                 ← full .git directory
├── src/
└── ...
~/projects/hotfix/        ← worktree as sibling
~/projects/feature-x/     ← another worktree as sibling
```

`.git` lives inside the original clone. Every `git worktree add` creates a sibling directory next to it.

This is the default, and plenty of heavy worktree users stay on this layout forever and never need anything more.

### Bare clone with child worktrees

An alternative layout that really shines when you work on two or more branches in parallel regularly:

```treeview
~/projects/my-repo/
├── .bare/                ← the bare repo (objects, refs, worktree registry)
│   ├── config
│   ├── HEAD
│   ├── objects/
│   ├── refs/
│   └── worktrees/
├── .git                  ← 16-byte pointer FILE: `gitdir: ./.bare`
├── main/                 ← worktree for branch `main`
├── dev/                  ← worktree for branch `dev`
└── feature-x/            ← worktree for branch `feature-x`
```

Every branch you want to work on becomes a directory, and all of them are children of the repo dir. There's no "primary" working tree; `.bare/` holds the shared state.

Properties worth knowing:

- The repo-root `.git` is a **file**, not a directory. Its entire contents are `gitdir: ./.bare`. Git commands run from a child worktree follow this pointer to reach `.bare/`.
- Running `git status` from the repo root itself fails with "this operation must be run in a work tree". That's expected. You always work from inside one of the child worktrees.
- Local branch refs (`refs/heads/*`) only exist for branches that have a worktree checked out. Remote-tracking refs exist for every branch on the remote.

### When to pick each

- **Regular clone** for repos you mostly touch one branch at a time, or small repos where the worktree mental model isn't worth it.
- **Bare clone with child worktrees** for repos where you regularly have two or more branches in flight — long-lived feature plus `main` ready for hotfixes, or active PRD work plus a clean `main` you can always build against.

## The setup gotcha nobody warns you about

If you try to build the bare-clone layout by hand, you'll hit this: **`git clone --bare` does NOT configure the remote-tracking refspec.**

The bare repo's `config` ends up with no `fetch = +refs/heads/*:refs/remotes/origin/*` line, which means `git branch -a` won't show remote branches, and `git fetch` won't populate them either. This is the single most common reason "my worktrees are broken" threads exist.

Fix it with `git config` inside the bare repo:

```bash
git --git-dir=.bare config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
git --git-dir=.bare fetch
```

Or use the alias below, which does it automatically.

## My aliases

I've wrapped both setup steps in global git aliases so I don't have to remember the details each time.

**`git clone-wt <url> [dir]`** sets up the bare-clone layout from scratch:

```bash
git clone-wt ssh://git@example.com/org/my-repo.git
# creates ./my-repo/ with .bare/, .git pointer, and a main/ (or default-branch) worktree
```

It performs:

1. `git clone --bare <url> .bare`
2. Writes the `.git` pointer file (`gitdir: ./.bare`)
3. Rewrites `remote.origin.fetch` to `+refs/heads/*:refs/remotes/origin/*`
4. Sets `core.repositoryformatversion = 1` and `extensions.worktreeConfig = true` so per-worktree config scoping is available out of the box (see the shared-config gotcha below)
5. Fetches from origin
6. Detects the default branch via `symbolic-ref HEAD` on the bare repo (works for `main`, `master`, `trunk`, whatever)
7. Deletes stray local refs except the default branch
8. Adds the default branch as a child worktree named after the branch

**`git new-wt <dir-name> [branch] [base-branch]`** adds a new worktree with sensible defaults:

```bash
git new-wt prd-42-auth-rewrite
# creates ../prd-42-auth-rewrite with a new branch feature/prd-42-auth-rewrite off main
```

Defaults: branch = `feature/<dir-name>`, base = `main`. It always places the new worktree at `../<dir-name>` relative to the current worktree, which is the correct spot in both layouts (sibling for regular clones, child-of-repo-dir for the bare-clone layout).

Here are the actual alias definitions. Add them to your `~/.gitconfig` under `[alias]` (or use `git config --global alias.<name> '<body>'`):

<details>
<summary><strong><code>git clone-wt</code></strong> — the full shell body</summary>

```bash
!f() {
  set -e
  url="$1"
  dir="${2:-$(basename "$url" .git)}"
  if [ -e "$dir" ]; then
    echo "error: $dir already exists" >&2
    return 1
  fi
  mkdir -p "$dir"
  cd "$dir"
  git clone --bare "$url" .bare
  printf "gitdir: ./.bare\n" > .git
  git --git-dir=.bare config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
  git --git-dir=.bare config core.repositoryformatversion 1
  git --git-dir=.bare config extensions.worktreeConfig true
  git --git-dir=.bare fetch origin
  default_branch=$(git --git-dir=.bare symbolic-ref --short HEAD)
  for ref in $(git --git-dir=.bare for-each-ref --format="%(refname:short)" refs/heads/); do
    if [ "$ref" != "$default_branch" ]; then
      git --git-dir=.bare branch -D "$ref" >/dev/null
    fi
  done
  git --git-dir=.bare worktree add "$default_branch" "$default_branch"
  echo ""
  echo "Layout ready at: $(pwd)"
  echo "  .bare/                (the bare repo)"
  echo "  .git                  (pointer file)"
  echo "  $default_branch/      (worktree for $default_branch)"
  echo ""
  echo "Start working:  cd $dir/$default_branch"
}; f
```

</details>

<details>
<summary><strong><code>git new-wt</code></strong> — the full shell body</summary>

```bash
!f() {
  set -e
  name="$1"
  branch="${2:-feature/$name}"
  base="${3:-main}"
  if [ -z "$name" ]; then
    echo "usage: git new-wt <dir-name> [branch] [base-branch]" >&2
    return 1
  fi
  git worktree add -b "$branch" "../$name" "$base"
  echo ""
  echo "Worktree created. Run this to start working:"
  echo "  cd ../$name && claude"
}; f
```

</details>

{{< callout type="info" >}}
If you'd rather not hand-roll aliases, there's also [worktree-manager](https://github.com/jarredkenny/worktree-manager) — a dedicated CLI for creating, listing, and cleaning up worktrees. I'm not using it myself, but it looks interesting and might be worth a try.
{{< /callout >}}

## How I use worktrees with AI

### A worktree per PRD or big task

Whenever I start a [PRD]({{< ref "ai-tips" >}}#write-prds-clear-sessions-often) or any large piece of work, I branch it into a worktree. `main` stays clean and buildable the whole time. If the experiment doesn't pan out:

```bash
# must cd out first — git worktree remove refuses if you're inside the target
cd ../main
git worktree remove ../prd-42-auth-rewrite
git branch -D feature/prd-42-auth-rewrite
```

Gone. No cherry-picking, no reverting, no surgical undo.

### Parallel Claude Code sessions

I typically run two or three Claude Code sessions at once (see [Run Multiple Sessions]({{< ref "ai-tips" >}}#run-multiple-sessions--the-dialup-workaround)). Each session gets its own worktree, so they never step on each other's files.

A typical setup:

- **Tab 1:** `prd-47-payments/` worktree, working through the current PRD milestone
- **Tab 2:** `fix-auth-regression/` worktree, investigating a bug
- **Tab 3:** `refactor-config-loader/` worktree, exploring a refactor

Objects are shared across all three, but the checkouts themselves aren't: you pay disk for each worktree's source tree, not for three full repos. For a 2 GB source × 3 worktrees, that's ~6 GB of checkouts plus a single shared `.git/objects`. Switching between them is just switching terminal tabs.

### Cheap experiments

Sometimes you just want to try something risky: upgrade a major dependency, rewrite a module, apply an AI-proposed refactor that looks plausible but might be junk. Worktree it:

```bash
git new-wt experiment-sqlc-upgrade
cd ../experiment-sqlc-upgrade
# let AI rip
```

If it works, merge it. If it doesn't, delete it. The experiment never touched `main` and never touched any of your other in-flight work.

{{< callout type="info" >}}
Worktrees pair directly with the **[Keep `main` Safe]({{< ref "ai-tips" >}}#keep-main-safe)** discipline: branch before you prompt, throw away cheaply when things go sideways.
{{< /callout >}}

## Gotchas

- **You can't check out the same branch in two worktrees at once.** Git refuses. Use `-b` to create a new branch off it instead.
- **Bare-clone refspec.** Covered above. Use `git clone-wt` or set `fetch = +refs/heads/*:refs/remotes/origin/*` manually on the bare repo's config.
- **`git status` at the bare-clone root fails.** That's not a bug; the root itself isn't a worktree. Always `cd` into a child worktree first.
- **Stale local branches** can accumulate after worktrees are removed. `git worktree prune` cleans up worktree metadata; `git branch -d` removes the branch ref if you want.
- **Most config is shared across worktrees.** By default, `git config --local <key>` from any worktree writes to the shared repository config, not anything worktree-specific. Per-worktree scoping exists via `git config --worktree`, but it's opt-in: the repository needs `extensions.worktreeConfig = true` (which requires `core.repositoryformatversion = 1`). The `clone-wt` alias above sets both during setup.
- **Keeping worktrees up to date.** `git fetch` updates remote-tracking refs repo-wide once, but each worktree still needs its own `git pull` or `git rebase origin/<branch>` to move its working tree.

## Further reading

- [Official `git worktree` docs](https://git-scm.com/docs/git-worktree) — the reference
- [Pablo Arias — Boost Your Workflow: Exploring Git Worktrees](https://pabloariasal.github.io/2023/12/27/git-worktrees/) — friendly intro
- [matklad — How I Use Git Worktrees](https://matklad.github.io/2024/07/25/git-worktrees.html) — a Rust perspective
- [incident.io — Shipping faster with Claude Code and Git Worktrees](https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees) — a production team's take on the AI-pairing angle

