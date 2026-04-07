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
- One active product-level quick at a time

**Milestone-context quick** (active milestone, no `--main`):
- Runs inside the existing milestone worktree on the milestone branch
- No separate worktree or branch — changes merge when the milestone completes
- No `complete-quick` or `abandon-quick` needed

The `--full` and `--debug` flags change workflow guidance (tests expected, investigation focus) but git mechanics are identical across all flavors.

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

One active product-level quick at a time. If you need parallel work, that's what milestones are for.

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
*v19.0 — Git Worktrees for Code Repo Isolation*
