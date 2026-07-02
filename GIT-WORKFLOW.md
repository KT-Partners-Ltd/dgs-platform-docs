# How Git is Used in DGS

A conceptual overview of how DGS manages git operations — work modes, worktrees, merging, and setup. For command reference, see the [User Guide](USER-GUIDE.md).

---

## The Three Work Modes

DGS provides three modes for making changes. Each mode manages git differently based on the scope of work.

| Mode | When to Use | Creates Worktree? | Branch | Merges Via |
|------|------------|-------------------|--------|------------|
| Fast (`dgs:fast`) | Trivial 1-10 line fixes: typos, config tweaks | No | Direct to `base_branch` | Immediate commit |
| Quick (`dgs:quick`) | Bug fixes, small contained changes | Product-level: yes. Milestone-context: no | `quick/{title}` or milestone branch | `dgs:complete-quick` (rebase + merge) |
| Milestone (`execute-phase`) | Planned multi-phase work | Yes (on first execute-phase) | `milestone/{slug}` | `dgs:complete-milestone` (rebase + merge) |

> When `git.completion_mode: pr` is set, the "Merges Via" completion commands open a GitHub pull request instead of merging locally — see [Completion Modes: Merge vs PR](#completion-modes-merge-vs-pr).

### Decision Flow

- **Trivial fix** (typo, config tweak) → `dgs:fast`
- **Bug during milestone work, related to milestone** → `dgs:quick` (runs in milestone worktree, no new branch)
- **Bug during milestone work, unrelated** → `dgs:quick --main` (creates separate worktree off main)
- **Bug with no active milestone** → `dgs:quick` (creates worktree automatically)
- **Planned feature work** → milestone via `execute-phase`

### Two Flavors of Quick

**Product-level quick** (no active milestone, or `--main` flag):
- Creates an ephemeral worktree off main with a `quick/{title}` branch
- Full lifecycle: `dgs:complete-quick` to merge, `dgs:abandon-quick` to discard
- Up to **3 standalone quicks** can be live in parallel, each with its own worktree and branch — the cap counts quick worktrees only; an active milestone never consumes a slot

**Milestone-context quick** (active milestone, no `--main`):
- Runs inside the existing milestone worktree on the milestone branch
- No separate worktree or branch — changes merge when the milestone completes
- No `complete-quick` or `abandon-quick` needed

The `--full` and `--debug` flags change workflow guidance (tests expected, investigation focus) but git mechanics are identical across all flavors.

### The Base Model: Inline vs `--main`

Every piece of work has a **base** — the branch its changes ride on. DGS resolves it with one rule: **inline by default**.

- **Inline (default):** new work joins whatever is already in flight in the console's focused context. A quick started while a milestone is focused runs *inside* the milestone worktree, on the milestone branch — no new worktree, no separate completion. A `/dgs:fast` on a focused standalone quick inlines onto that quick's worktree.
- **`--main`:** forces a separate, product-level worktree cut from `git.base_branch`, regardless of any active milestone. Use it for work that must not ship with (or wait for) the milestone — it gets its own `quick/{slug}` branch and its own completion lifecycle.

The `--main` mentions elsewhere ([Command Reference](COMMAND-REFERENCE.md), [Milestone Jobs Guide](MILESTONE-JOBS-GUIDE.md)) all refer to this model. How the *completion* of that work then lands — local merge or pull request — is a separate, orthogonal switch: [Completion Modes: Merge vs PR](#completion-modes-merge-vs-pr).

---

## Worktree Lifecycle

A git worktree is a second checkout of the same repository in a different directory. DGS uses worktrees so the main checkout stays clean and available for fast fixes while longer-running work happens elsewhere.

### Directory Layout

```
~/dev/
├── myapp/                          <- main checkout (always on main, always clean)
│   └── ...
├── myapp--gsd-v19/                 <- milestone worktree (on milestone/v19 branch)
│   └── ...
└── myapp--gsd-quick-fix-auth/      <- quick worktree (on quick/fix-auth branch)
    └── ...
```

Worktrees are siblings to the main checkout. The naming convention is `{repo}--{project_slug}-{milestone_or_quick_slug}`.

### Lifecycle Stages

1. **Created** — `execute-phase` (milestone) or `dgs:quick` (product-level) creates the worktree automatically on first use.
2. **Active** — Work happens in the worktree. Commits go to the worktree's branch. The main checkout is untouched.
3. **Completed** — `complete-milestone` or `complete-quick` rebases, merges to main, removes the worktree and branch.

For milestones, the worktree persists across all phases. It is created on the first `execute-phase` and removed by `complete-milestone`.

For product-level quicks, the worktree is ephemeral. Created by `dgs:quick`, removed by `complete-quick` or `abandon-quick`.

Up to 3 standalone quicks can be live at once, each in its own worktree. See [Parallel Consoles & Focus](#parallel-consoles--focus) for how each console keeps its own focus among them.

---

## Rebase-Before-Merge Strategy

DGS uses rebase-before-merge for all completion workflows. This produces a clean linear history with no merge commits.

### Step-by-Step Flow

Both `complete-quick` and `complete-milestone` follow the same sequence:

```
1. Pull latest main          git fetch origin && git pull origin main
2. Rebase in worktree        git -C {worktree} rebase main
3. If conflicts              → conflict-agent attempts auto-resolution
                             → if it can't: abort rebase, show manual instructions
4. Fast-forward merge        git merge --ff-only {branch}
5. Push                      git push origin main
6. Cleanup                   remove branch + worktree
```

### What the History Looks Like

```
Before rebase:
main:      A---B---C
                \
milestone:       D---E---F

After rebase + ff-merge:
main:      A---B---C---D'---E'---F'
```

The result is a single straight line. No merge commits, no tangled history.

### Conflict Handling

When rebase encounters conflicts:

1. DGS's conflict-agent tries to resolve automatically, processing each commit during the rebase one at a time.
2. If it cannot resolve: the entire rebase is aborted (`git rebase --abort`), leaving the worktree in a clean pre-rebase state.
3. DGS provides copy-paste commands for manual resolution:

```bash
cd ~/dev/myapp--gsd-v19          # cd to worktree
git rebase main                  # start rebase
# resolve conflicts, then:
git add .
git rebase --continue
# repeat if multiple commits have conflicts
```

4. After manual resolution, re-run `complete-milestone` or `complete-quick`. It detects the rebase is already done and skips straight to the fast-forward merge.

> **Note:** During rebase, "ours" refers to the working branch and "theirs" refers to main. This is the opposite of the merge perspective.

---

## Completion Modes: Merge vs PR

Both `complete-quick` and `complete-milestone` honor one config key: `git.completion_mode`.

| Value | What completion does |
|-------|----------------------|
| `merge` (default — an absent key means `merge`) | The rebase-before-merge flow above: rebase, fast-forward merge to `base_branch`, push, remove worktree and branch |
| `pr` | Rebase, push the branch, open (or update) a GitHub pull request — then stop. Nothing merges locally; the merge happens on GitHub |

Set it in the tracked `config.json`:

```
dgs-tools config set git.completion_mode pr
```

The value is enum-validated — anything other than `merge` or `pr` is rejected.

### The PR Flow (`completion_mode: pr`)

PR completion forks off the same rebase prefix the merge path uses, then:

1. **`gh` preflight** — verifies the remote is a GitHub host and the GitHub CLI (`gh`) is installed and authenticated, *before* anything is pushed.
2. **Leased push** — the work branch (`quick/{slug}` or `milestone/{slug}`) is pushed with `git push --force-with-lease` (never plain `--force`), so a rebase-rewritten branch updates its remote without being able to clobber unseen remote work.
3. **Idempotent PR open/update** — guarded by `gh pr list --head`: the first run creates the PR (`gh pr create`, seeding a title and a commit-list body); re-runs push the new head only and never overwrite a human-edited PR title or body.
4. **Stop.** No fast-forward merge, no push to `base_branch`, no teardown. The worktree and branch stay until the PR merges.

DGS records what it opened on the worktree entry — per repo: `pr_number`, `pr_url`, and `pr_head_sha` (the exact head that was pushed) — and flips the entry to `state: pr_open`.

> **`gh` is required only in PR mode** (and for fast-PR). The default `merge` path never invokes the GitHub CLI. If the preflight fails, the error says how to install or authenticate `gh` — or to set `git.completion_mode` back to `merge`.

### The Open → Reap State Machine

A `pr_open` quick or milestone is parked, not finished. Re-running its completion command drives the state machine:

- **All PRs merged** → **reap**: pull `base_branch`, remove the worktree and branch, drop console bindings pointing at the slug. For milestones, archival happens here — at reap, never at open — so a closed-unmerged PR can never leave a prematurely-archived milestone.
- **A PR still open** → **update**: new commits are pushed to the same PR head.
- **A PR closed without merging** → completion refuses (your work is unmerged); re-run with `--confirm-cleanup` to remove the worktree — nothing is merged.
- **`gh` outage** → fails closed. A `gh` failure is never interpreted as "not merged"; you get `Couldn't reach GitHub. If you know it merged, re-run with --merged`.
- **`--merged` escape hatch** — asserts the merge and reaps *without* `gh`. Valid only from `pr_open`. `dgs-tools reap-quicks` runs the same merged-quick check as a sweep across every live quick — see the [Command Reference](COMMAND-REFERENCE.md).

Milestone-specific: four-eyes governance gates at PR *open* only; the post-merge reap re-run needs no re-approval.

### Multi-Repo PR Tracking (`entry.prs`)

A multi-repo quick or milestone opens **one PR per touched repo**, and each repo keeps its own record on the worktree entry's `entry.prs` map:

```json
"prs": {
  "api-service": { "pr_number": 41, "pr_url": "https://github.com/...", "pr_head_sha": "..." },
  "web-app":     { "pr_number": 17, "pr_url": "https://github.com/...", "pr_head_sha": "..." }
}
```

- Each repo's record is persisted independently, under the `__config__` mutex (see [Concurrency & Locking](#concurrency--locking)) — opening repo B's PR can never clobber repo A's record.
- Merge detection gates on **all** repos: the entry reaps only when *every* repo's PR is merged. A partial merge never reaps.
- The pre-reap "post-merge work" guard compares each repo against its **own** `pr_head_sha`, so commits made after the last push are caught per repo.

---

## Parallel Consoles & Focus

Parallel work is per-console: each console (Claude Code session or raw shell) resolves its **own** active context — the milestone or standalone quick its commands operate on — with this precedence, highest first:

1. `--context <slug>` — a one-shot per-command flag
2. Session binding — `CLAUDE_CODE_SESSION_ID` → `config.local.json` `execution.console_bindings` (the default path inside Claude Code; no shell setup)
3. `DGS_CONTEXT` — the per-shell environment variable (raw-shell path)
4. The config default (`execution.active_context`)
5. None (product / main)

Two consoles on the same project can therefore each drive a different milestone or quick: commit routing, completion, and the status bar all follow the console's own focus. The [Concurrency & Locking](#concurrency--locking) layer below is what makes this safe — concurrent writes to the shared config store and planning repo are serialized and never lost. For setup and day-to-day use (binding a console, one-shot overrides, checking a binding), see [Per-Console Context](USER-GUIDE.md#per-console-context) in the User Guide.

---

## Concurrency & Locking

Parallel work is normal in DGS — multiple Claude Code consoles, parallel quick tasks, multi-repo milestones. All of it shares two pieces of state in the planning repo: `config.local.json` (worktree entries, focus/context bindings, execution locks, PR records) and the planning repo's git index (STATE.md, roadmap, plan artifacts). Since v25.2, a cross-process locking layer serializes writers of both, so concurrent commands cannot corrupt shared state. This section describes that layer at the level you can observe and rely on, and how to recover a stuck run.

### The Lock Substrate

The substrate is one primitive (in `config-lock.cjs`): an O_EXCL sentinel file — `.dgs-lock-<key>.sentinel`, created next to `config.local.json` — keyed by mutex name. Creating the file is atomic (`O_CREAT | O_EXCL`), so exactly one process can hold a given key at a time; the sentinel is always removed when the critical section ends. No external locking dependencies are involved.

There are two mutex keys, and they are never nested:

| Key | Guards | Typical hold |
|-----|--------|--------------|
| `__config__` | Every read-modify-write of `config.local.json` | Sub-millisecond |
| `__planning_git__` | The planning repo's `git add` + `git commit` (and STATE.md read-modify-write) critical sections | Commit-duration (seconds) |

Config writes go through the guarded `mutate(cwd, '__config__', fn)` primitive: the whole config file is read fresh **under the lock**, the caller mutates only its own subtree in place, and the whole freshly-read object is written back, still under the lock. This field-level merge means two processes writing different subtrees — say, one binding a console while another records a worktree entry — can never lost-update each other; the classic read-then-write window is closed. Git critical sections use `withLock(cwd, '__planning_git__', fn)`, a pure mutual-exclusion section that never touches the config file (so a multi-second commit never blocks config writers).

Lock acquisition is bounded, never blocking-forever: a single attempt retries the sentinel for roughly half a second, and callers that need to ride out momentary contention retry the whole operation up to 6 times before **failing loudly with a lock-contention error** stating that the write did not land. A caller never hangs indefinitely and never proceeds unlocked.

**Stale-reclaim (per key).** A sentinel leaked by a crashed holder would otherwise wedge its key forever, so each key has its own stale-reclaim threshold: a `__config__` sentinel older than **5 seconds** is reclaimed (those writes are sub-millisecond, so anything older was leaked), while a `__planning_git__` sentinel gets **60 seconds** — a legitimately slow multi-second `git commit` is never mistaken for a leak and never reclaimed out from under itself. These are deliberately fixed thresholds rather than a sentinel heartbeat: a crash self-heals after the window, with no moving parts. (A heartbeat does exist one layer up, on the longer-lived execution lock — see below.)

### What Each Mutex Covers

**The config store.** Every DGS writer of `config.local.json` — worktree entries, active-context and console bindings, execution locks, fast-PR records, per-repo PR tracking — routes its read-modify-write through the guarded mutator under `__config__`. No unlocked writer of that file remains.

**The planning-repo git mutex.** Every DGS committer to the planning repo — the quick, milestone, and phase artifact committers, the STATE.md writers, and the generic commit command — stages and commits under `__planning_git__`, so two committers operating in the same single planning checkout can no longer interleave their `git add` / `git commit` steps.

**Path-limited commits.** Committers do not commit the whole index. Each commit is path-limited to exactly the pathspecs that committer just staged, via `git commit -m <msg> -- <paths>`. Even if a lock were somehow bypassed, one writer's commit cannot sweep another writer's staged-but-uncommitted files into the wrong commit.

**`git push` happens outside the lock.** The mutex covers only the local critical section (stage + commit). A network push is never performed while holding `__planning_git__`, and a push only runs after a commit has actually landed. Pushes are therefore not serialized by this layer — only local history writes are.

### Guarantees You Can Rely On

These are behavioral guarantees proven by multi-process contention tests (below), not aspirations:

- **Contended writes never report false success.** If a write cannot take its lock within the retry budget, the operation fails visibly — a structured contended result or a thrown `DGS lock contention` error stating the write did NOT land. You will never see `updated: true`, `created: true`, or `committed: true` for a write that did not happen, and a write is never silently dropped.
- **Exactly one active milestone survives a race.** Milestone creation persists its entry inside the `__config__` mutex and re-checks, under that same lock, that no different milestone won in the meantime (closing the check-then-write race). The loser is fully rolled back — its just-created git worktrees, branch, and config entry are removed — and it reports: `A different milestone (<winner>) won the race; rolled back <slug>. Retry.` Re-creating the same slug is idempotent, and `--force` deliberately writes through.
- **Quick/fast task IDs are collision-proof.** IDs (`YYMMDD-xxx`) draw their 3-character suffix from a per-process, cryptographically seeded monotonic counter — not the clock — so same-instant parallel creates within a process can never collide, and cross-process collisions are improbable rather than guaranteed by a shared time bucket.

**What the tests prove.** Two fork-based harnesses back these claims with real multi-process races. `concurrency-race.test.cjs` proves the substrate primitives: 200 concurrent config writes across two processes lose zero entries, concurrent ID draws contain zero duplicates, exactly one milestone entry survives a creation race, concurrent planning-repo commits each contain only their own files, and the per-key stale thresholds hold (a live slow commit is not reclaimed; a leaked config sentinel is). `concurrency-caller-race.test.cjs` goes further and drives the **real command callers** — console binding, config-set, fast-PR record, the generic committer, milestone worktree creation, PR-record persistence — under two-process contention with no test-side retry wrapper, proving each caller surfaces contention as a failure (an explicit error or `contended: true` result) instead of false success or a silent drop.

### Recovering a Stuck Run: the Execution Lock

Separate from the sub-second sentinel mutexes above, DGS keeps a longer-lived **execution lock**: one executor per milestone worktree at a time. The lock record is `execution.executing.<slug> = { started_at, session_id }` in `config.local.json` (its own reads and writes run under `__config__`, so acquiring it is race-free). If a second executor tries to start against the same worktree while the lock is live, it is refused and told which session holds it.

- **6-hour stale escape.** A lock entry older than 6 hours is presumed crashed: the next acquire warns (`ignoring a stale executor lock`) and takes over. A crashed run never wedges its worktree forever.
- **Heartbeat for long runs.** A healthy multi-wave execution can legitimately exceed 6 hours. `execute-phase` therefore re-stamps the lock at every wave boundary — a same-session re-acquire is an idempotent `started_at` refresh — so staleness is measured from the last wave boundary, not the run start, and a live long run is never falsely reclaimed.
- **Manual recovery.** If a run is genuinely dead but its lock is younger than 6 hours, release it explicitly:

  ```bash
  dgs-tools execution-lock release <slug> --force
  ```

  `execution-lock release` is session-aware: without `--force` it refuses to free a live lock belonging to a different session (returning `released: false` plus the holder's session id). `--force` overrides, and a stale (>6h) entry is always releasable. `acquire` accepts `--force` too — it warns and proceeds, never silently.

**Two staleness layers — don't confuse them:**

| Layer | Threshold | What staleness means |
|-------|-----------|----------------------|
| Sentinel stale-reclaim (`__config__` / `__planning_git__`) | 5s / 60s | A leaked lock *file* from a crashed process; self-heals automatically |
| Execution-lock staleness | 6 hours | A crashed *run*; auto-released on the next acquire, or freed manually with `release --force` |

---

## Setup Commands & Monorepos

REPOS.md has an optional `setup` field per repo. DGS runs this command whenever it creates a worktree, handling dependency installation and environment preparation automatically.

### REPOS.md Setup Field

```markdown
| Name | Path | Setup |
|------|------|-------|
| api-service | ../api-service | npm install |
| web-app | ../web-app | ./scripts/setup-worktree.sh |
```

The setup command receives:

- **$1** — the milestone or quick slug (e.g., `v19`, `fix-auth`)
- **$2** — the absolute path to the worktree directory
- **cwd** is set to the worktree directory
- **Timeout** — 5 minutes

### Simple Node.js Project

```
setup: npm install
```

### npm/pnpm Monorepo

```bash
#!/bin/bash
# scripts/setup-worktree.sh
SLUG=$1
WORKTREE_PATH=$2

# Install all workspace dependencies
npm install

# Build shared packages that other packages depend on
npm run build --workspace=packages/shared
```

For pnpm workspaces:

```bash
#!/bin/bash
pnpm install --frozen-lockfile
pnpm -r --filter './packages/shared' build
```

The worktree is created at the repo level. For monorepos, the setup script handles internal topology — workspace hoisting, selective builds, symlinks. DGS has no monorepo-specific logic.

### Setup Failures

If setup fails, the worktree remains in valid git state. Fix the issue and re-run:

```
dgs-tools worktrees setup {slug}
```

### Useful Commands

```
dgs-tools worktrees list              # Show all active worktrees
dgs-tools worktrees setup {slug}      # Re-run setup for a worktree
dgs-tools worktrees prune             # Clean up orphaned worktree entries
```

For command details, see the [User Guide](USER-GUIDE.md).

---
*The DGS git model — worktree isolation, rebase-before-merge, PR & merge completion, parallel consoles, and the concurrency/locking layer.*
