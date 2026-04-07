# Multi-User Guide for DGS

How to use Deliver Great Systems effectively when multiple people are working on the same product.

DGS is designed for teams building products where the primary effort is developing specifications — capturing ideas, refining them through discussion and research, and producing detailed specs that feed into the execution pipeline. The execution pipeline is serialised by default, but since AI-assisted code generation has reduced build time to a small fraction of what it was, this is no longer a bottleneck. The team's time is best spent on the upstream work — ideas, specs, and design — which is naturally parallel.

This guide covers what works safely in parallel, what doesn't, and how to configure DGS for team use.

---

## The Core Rule

**One executor per phase at a time.**

DGS tracks state in shared files (STATE.md, ROADMAP.md, manifest counters). There are no file locks. Two people executing the same phase will corrupt these files. Most of the advice in this guide follows from this constraint.

---

## Getting Started for Teams

Step-by-step setup for a team adopting DGS on a shared product.

### 1. Initialise the product

```bash
/dgs:init-product
```

This creates the `.planning/` directory structure and `config.json` with team-ready defaults.

### 2. Register code repositories

```bash
/dgs:add-repo api-service ../api-service
/dgs:add-repo web-app ../web-app
```

Register every code repo the product touches. DGS will sync all of them.

### 3. Choose a branching strategy

| If your team... | Use | Why |
|-----------------|-----|-----|
| Wants PRs per phase | `phase` | Finest isolation, one branch per phase per project |
| Wants PRs per release | `milestone` | One branch per milestone, fewer merge points |
| Works solo or trusts main | `none` | Simplest, all commits to current branch |

```bash
dgs-tools config-set git.branching_strategy phase
```

For teams, `phase` is recommended -- it gives each phase its own branch for independent code review.

### 4. Choose a sync mode

| If your team... | Set sync to | Why |
|-----------------|-------------|-----|
| Is new to DGS and wants control | `prompt` (default) | See and approve each sync operation |
| Has established workflows and wants hands-off sync | `auto` | Silent sync, report only failures |
| Prefers manual git operations | `off` | DGS never touches remotes |

```bash
# Set both push and pull at once
dgs-tools config-set git.sync prompt

# Or set independently
dgs-tools config-set git.sync_push auto
dgs-tools config-set git.sync_pull prompt
```

Fresh installs default to `prompt`. If your team prefers different sync behaviour for push vs pull (e.g., auto-push but prompt-pull), set them independently.

### 5. Configure user identity

Each team member should set their identity on their machine:

```bash
/dgs:set-profile
```

This ensures ideas, specs, and plans are attributed to the correct author.

### 6. Agree on project ownership

- **Ideas and specs:** Anyone can add at any time -- file-per-entity design prevents conflicts
- **Phase execution:** One person per phase at a time (the core rule)
- **Quick tasks:** Only when no phase is actively executing

### 7. Verify the setup

```bash
/dgs:health
```

Confirms repos are registered, config is valid, and git state is clean.

---

## What's Safe in Parallel

### Ideas and Spec Development

The ideas pipeline is the most multi-user-friendly part of DGS. Ideas are stored as individual files with auto-incremented IDs, and specs are similarly isolated.

**Safe parallel activities:**

| Activity | Why it works |
|----------|-------------|
| Two people adding ideas (`/dgs:add-idea`) | Each creates a separate file in `pending/` |
| Discussing different ideas (`/dgs:discuss-idea`) | Each appends to its own idea file |
| Researching different ideas (`/dgs:research-idea`) | Each writes to its own idea file |
| Developing different ideas (`/dgs:develop-idea`) | Same — isolated per idea file |
| Writing specs from different ideas (`/dgs:write-spec`) | Each spec gets its own file |
| Refining different specs (`/dgs:refine-spec`) | Edits are to separate spec files |

**One caveat:** The `manifest.json` counter (`next_id`) is not locked. If two people run `/dgs:add-idea` at the exact same moment, they could get the same ID. In practice this is rare — it requires near-simultaneous execution — but if it happens, one idea overwrites the other. The simple fix is communication: "I'm adding an idea now."

**Recommended workflow for teams:**

1. Anyone can add ideas at any time — the collision risk is low enough to ignore
2. Assign spec ownership — one person drives a spec from write through refine to approve
3. Cross-LLM review (`/dgs:write-spec` with OpenAI/Gemini reviewers configured) works the same regardless of who triggers it
4. Use `/dgs:approve-spec` as the handoff point — once approved, the spec is ready for milestone planning

### Discuss Phase

The `/dgs:discuss-phase` command gathers design context before planning. It writes only to a CONTEXT.md file in the phase's own directory — it does not modify ROADMAP.md, STATE.md, or any shared state.

**Discuss all phases upfront before planning or executing any of them.** This is not just safe — it's the recommended approach. Each `/dgs:discuss-phase` produces a CONTEXT.md that captures your design decisions, implementation preferences, and phase boundaries. The planner reads this context to produce plans that reflect your intent rather than guessing.

This is especially important for job execution (`/dgs:create-milestone-job` + `/dgs:run-job`). Jobs run planning and execution automatically without pausing for interactive input. If a phase has no CONTEXT.md, the planner will work from requirements and research alone — your design preferences won't be included. Discussing all phases before creating a job ensures every phase in the milestone has the context it needs.

**Recommended milestone kickoff:**

```
/dgs:discuss-phase 1
/dgs:discuss-phase 2
/dgs:discuss-phase 3
...
/dgs:discuss-phase N

# Then plan and execute (or create a job)
/dgs:create-milestone-job
/dgs:run-job
```

**How to use it in a team:**

- Multiple team members can discuss the same phase at different times — sessions append to the discussion log
- The context is consumed later during planning — it doesn't modify shared state
- Think of it as a design review conversation that happens before any code
- CONTEXT.md can be updated later if decisions change — `/dgs:plan-phase` will offer to use or update existing context

### Different Phases (with branching)

Two people can work on different phases simultaneously **if you use the `phase` branching strategy**:

```json
{
  "git": {
    "branching_strategy": "phase"
  }
}
```

With phase branching, each `/dgs:execute-phase` creates its own Git branch (e.g., `dgs/phase-03-auth`, `dgs/phase-04-checkout`). The code changes are isolated. However, both phases still update the same STATE.md and ROADMAP.md files.

**Safe approach:**

1. Each person owns one phase at a time
2. Use phase branching so code doesn't conflict during development
3. Accept that STATE.md and ROADMAP.md will need manual merge-conflict resolution at branch merge time
4. Merge phases to main sequentially, not simultaneously

**The ROADMAP/STATE conflict problem:**

When Person A completes Phase 3 and Person B completes Phase 4, both branches contain updates to ROADMAP.md (checking off their phase's progress) and STATE.md (updating position). These will conflict when merging the second branch. The conflicts are straightforward to resolve — you want both sets of updates — but they require manual attention.

### Different Projects (v2 Multi-Project)

The v2 multi-project setup provides the strongest isolation. Each project has its own directory under `.planning/{project-slug}/` with separate STATE.md, ROADMAP.md, and phase directories.

```
.planning/
  PROJECTS.md
  REPOS.md
  project-alpha/
    STATE.md
    ROADMAP.md
    phases/
  project-beta/
    STATE.md
    ROADMAP.md
    phases/
```

**Two people can execute phases in different projects simultaneously** — the state files are completely separate. The only shared files are PROJECTS.md and REPOS.md, which are rarely updated during execution.

**Git branching in multi-project mode:**

Use the `phase` branching strategy. Each project's phases get their own branches, and the project name is included in commit scopes (`feat(project-alpha/04-01): ...`), so branches and commits are clearly attributed. When both projects touch the same repo, their phase branches are independently mergeable — merge Project Alpha's phase branch first, then Project Beta's, resolving any file-level conflicts at that point.

With the `milestone` branching strategy, all phases within a project share one branch (e.g., `dgs/v1.0-alpha`). This still provides project-level isolation, but gives you fewer, larger merge points. Choose `milestone` if you prefer one PR per project release rather than one PR per phase.

With `none` (no branching), both projects commit directly to main. This works if the projects don't share repos, but creates interleaved commits that are harder to untangle if something goes wrong.

**Where it can still clash:** If both projects touch the same code repository and modify the same files. Use `/dgs:overlap-check` to identify these overlaps before starting work. If overlap is detected, coordinate which project changes which files, or accept that you'll need to resolve merge conflicts at merge time.

---

## What's NOT Safe in Parallel

### Same Phase Execution

Never have two people run `/dgs:execute-phase` on the same phase. Both would:

- Update STATE.md position (which task is current)
- Update ROADMAP.md progress checkboxes
- Write to the same SUMMARY.md
- Commit with overlapping phase-plan scopes

The result is corrupted state files, interleaved commits, and confused resume behavior.

### Quick Tasks During Phase Execution

The `/dgs:quick` command and `/dgs:execute-phase` can clash when run simultaneously, even by the same person. Here's why:

**Git tree conflicts:** Both commands commit to the working tree. If `/dgs:execute-phase` has staged files and `/dgs:quick` tries to commit, or vice versa, Git operations can fail or produce unexpected results.

**STATE.md updates:** Both commands update STATE.md. The phase executor writes position updates and metrics; the quick command writes to the "Quick Tasks Completed" table. Simultaneous writes corrupt the file.

**File overlap:** If a quick task touches files that the current phase is also modifying, you get uncommitted changes interfering with the phase executor's expectations.

**How to use quick tasks safely alongside phases:**

| Approach | Safety |
|----------|--------|
| Run quick task before starting phase execution | Safe |
| Run quick task after phase execution completes | Safe |
| Run quick task between plans (when phase is paused) | Safe |
| Run quick task on different repo than active phase | Usually safe |
| Run quick task while phase is actively executing | Not safe |

**If you must do urgent work during phase execution:**

1. Use `/dgs:pause-work` to pause the phase — this creates a `.continue-here.md` handoff and commits WIP state
2. Run your `/dgs:quick` task
3. Use `/dgs:resume-work` to pick up the phase where you left off

### Concurrent Config Changes

The `.planning/config.json` file has no locking. Don't run `/dgs:settings` simultaneously from two sessions. One will overwrite the other's changes.

---

## Git Strategy for Teams

### Parallelise Through Projects, Not Phases

Phases within a milestone are sequenced for a reason — later phases typically depend on what earlier phases built. Running Phase 3 and Phase 4 in parallel risks Phase 4 building against code that Phase 3 hasn't created yet, or both phases modifying the same files with conflicting assumptions.

**If you want to parallelise work across team members, use different projects.** The v2 multi-project setup gives each project its own state, roadmap, and phase directories. Two people can execute phases in different projects simultaneously with full isolation — provided the projects don't clash on the same code files.

Use `/dgs:overlap-check` before starting parallel project work to verify that projects aren't touching the same files in the same repos. If there is no overlap, parallel execution is safe. If there is overlap, coordinate which project owns which files, or accept sequential execution for the overlapping phases.

If you do end up with merge conflicts, DGS has a conflict resolution capability built into the merge-back workflow. When `git merge` encounters conflicts, a resolution agent examines each conflicted file alongside the plan metadata — it knows which tasks changed the file and why, reads the SUMMARY.md for each branch's intent, and classifies each conflict (additive, divergent, structural, or deletion). It resolves what it can with confidence and flags low-confidence resolutions for your review before committing. This is better than generic merge tools because the agent has the full context of what each branch was trying to achieve. That said, it's always better to avoid conflicts in the first place — the agent does its best, but intent-aware resolution is not the same as guaranteed correctness.

**Parallel phases within one project** should only be attempted if you are certain the phases have no file-level dependencies — for example, Phase 3 works entirely in `src/auth/` and Phase 4 works entirely in `src/payments/` with no shared modules. Even then, both phases update STATE.md and ROADMAP.md, which will need manual conflict resolution at merge time. In most cases, the time saved is not worth the coordination overhead.

### Choosing a Branching Strategy

| Strategy | Solo developer | Multi-project team | Parallel phases (if you must) |
|----------|------|------------------------------|----------------------|
| `none` (default) | Best | Works if projects don't share repos | Not recommended |
| `phase` | Good | Good — clear branch per phase per project | Required for isolation |
| `milestone` | Good | Good — one PR per project release | Not suitable |

For teams, use `phase` or `milestone` branching depending on whether you want PRs per phase or per release. With multi-project parallel work, `phase` branching is preferred — it gives the finest-grained isolation.

### Configuration

Set this in `.planning/config.json`:

```json
{
  "git": {
    "branching_strategy": "phase",
    "phase_branch_template": "dgs/phase-{phase}-{slug}",
    "milestone_branch_template": "dgs/{milestone}-{slug}"
  }
}
```

### Merge Discipline

When phase branches need to merge to main:

1. **Merge in phase order** — Phase 1 first, then Phase 2, etc. This respects the dependency order from the roadmap.
2. **Use squash merge** — Keeps main clean with one commit per phase. The detailed per-task history is preserved on the branch.
3. **Expect STATE.md/ROADMAP.md conflicts** — These are the planning files that both phases update. Resolution is mechanical: accept both sets of updates.
4. **Resolve conflicts promptly** — Don't let branches diverge far from main. Merge each phase as it completes rather than batching.

### Merge Conflicts

When merge conflicts occur, the DGS conflict resolution agent examines each conflicted file alongside the plan metadata — it knows which tasks changed the file and why, classifies each conflict by type, and resolves what it can. Low-confidence resolutions are flagged for your review before committing. In multi-repo setups, conflicts in one repo don't affect others — each repo merges independently.

**Best practice: avoid conflicts rather than resolving them.** This means:

- Use `/dgs:overlap-check` before starting parallel work on different projects
- Structure projects so they own distinct areas of the codebase
- Merge completed phases to main immediately rather than accumulating branches

### Rolling Back a Job

`/dgs:rollback-job` reverts all code changes made by a milestone job. When the job starts, DGS records the starting commit SHA for every registered code repo. On rollback, it runs `git reset --hard` to those SHAs in each repo, restoring the code to its exact pre-job state. Planning artifacts (STATE.md, ROADMAP.md, plans) are preserved — only code repos are reset. SUMMARY.md files for executed phases are deleted since they describe work that no longer exists.

**Implications for parallel work:**

Rollback is all-or-nothing across repos. If Project Alpha and Project Beta share the `api` repo, and you roll back Project Alpha's job, the `api` repo resets to its pre-job SHA — which also wipes any commits Project Beta made to that repo after Alpha's job started. This is the strongest argument for ensuring parallel projects don't share repos, or at minimum don't have overlapping execution windows.

Before rolling back:

- Ensure no other project has committed to the affected repos since the job started
- Check for uncommitted work in all affected repos — the hard reset will destroy it
- DGS shows a preview of which repos will be reset and requires you to type `"rollback"` to confirm

If you need to roll back one project's work while preserving another's, you'll need to use `git revert` manually on the specific commits rather than the job-level rollback.

### Push and Pull Cadence

DGS can automatically synchronise with remotes at workflow boundaries. Behaviour is controlled by two config keys — `git.sync_push` and `git.sync_pull` — each set to one of three modes:

| Mode | Behaviour | Recommended for |
|------|-----------|-----------------|
| `off` | No sync, no prompts — identical to pre-v17.0 | Solo developers, manual git users |
| `prompt` | Asks before pull/push with Yes as default | Teams starting with DGS (fresh install default) |
| `auto` | Silent sync, reports only failures | Established teams, CI/CD, job execution |

**What syncs:** Not every command triggers sync. A per-workflow cadence table determines which workflows pull, push, or both. The four patterns are:

| Cadence pattern | When it applies | Examples |
|-----------------|-----------------|----------|
| Pull + Push | Workflows that read shared state and produce artifacts | `execute-phase`, `plan-phase`, `write-spec`, `run-job` |
| Push only | Workflows that create artifacts but don't need latest state first | `pause-work`, `add-idea`, `import-spec`, `map-codebase` |
| Pull only | Workflows that consume state without producing shareable changes | `resume-work`, `progress`, `find-related-ideas` |
| No sync | Read-only or diagnostic commands | `help`, `list-ideas`, `search`, `debug` |

For the full table with per-command reasoning, see `references/sync-cadence.md`.

**The principle remains the same: push at DGS workflow boundaries, not on time intervals.** The difference is DGS now handles this automatically based on mode and cadence. Every command that produces a complete artifact — a spec, a CONTEXT.md, a PLAN.md, a SUMMARY.md — leaves the repo in a consistent state and is a safe push point.

**Safe push points:**

| When | Sync direction | Notes |
|------|:---:|-------|
| Before workflow starts | Pull | Ensures latest state |
| After workflow completes | Push | Shares completed artifacts |
| After each plan in execute-phase | Push (mid-workflow) | Incremental sharing, silent |
| After each phase in run-job | Push (mid-workflow) | Same |
| Mid-task during plan execution | Never | Repo may be in inconsistent state |

**Why mid-task pushes are unsafe:**

- **Planning repo:** STATE.md may say task 3 is in-progress but no SUMMARY.md exists yet. Another team member pulling this state sees an inconsistent picture.
- **Code repos:** A task may have committed several files but not yet reached a compiling or test-passing state. Pushing a broken branch triggers CI failures and confuses reviewers.

**Failure handling:** Pull failure aborts before DGS state changes — no work is lost and no state is corrupted. Push failure warns but does not abort; work is committed locally and can be pushed manually later. In `prompt` mode, pull failure offers to continue without pulling.

**Configuration:**

```bash
# Set both directions at once
dgs-tools config-set git.sync prompt

# Or set independently for different push vs pull behaviour
dgs-tools config-set git.sync_push auto
dgs-tools config-set git.sync_pull prompt

# Or use /dgs:settings for interactive toggle
```

---

## Multi-Repo Considerations

In v2 multi-project setups with multiple repositories:

### What Works Well

- Each repo gets independent commits scoped by project: `feat(project-name/04-01): description`
- Branch creation happens across all repos a phase touches
- Commit failures in one repo don't roll back others
- Each repo merges independently

### What Needs Coordination

- **Shared repos across projects:** If both Project Alpha and Project Beta touch the `api` repo, their phase branches will conflict when merged. Use `/dgs:overlap-check` to identify this upfront.
- **Cross-repo dependencies:** If Phase 3 in the `api` repo creates an endpoint that Phase 4 in the `web` repo consumes, these phases must execute sequentially, not in parallel.

---

## Recommended Team Workflows

### Small Team (2-3 people)

```
Person A: Ideas → Specs → Phase 1 execution
Person B: Ideas → Specs → Phase 2 execution (after Phase 1 dependencies met)
Person C: Quick tasks, bug fixes, documentation

Config:
  branching_strategy: "phase"
  commit_docs: true
  git.sync: "prompt"
```

**Rules:**
1. One person executes a phase at a time
2. Anyone can add ideas and discuss at any time
3. Quick tasks only when no phase is actively executing (or on a different repo)
4. Merge phase branches to main via PR before starting dependent phases

### Multi-Project Team

```
Person A: Owns Project Alpha — full lifecycle
Person B: Owns Project Beta — full lifecycle
Person C: Shared quick tasks, cross-project fixes

Config:
  branching_strategy: "phase"
  Multi-project v2 with PROJECTS.md and REPOS.md
  git.sync: "auto"
```

**Rules:**
1. Each person owns their project's phases
2. Run `/dgs:overlap-check` before each milestone to identify shared-repo conflicts
3. When projects share repos, coordinate merge order
4. Use `/dgs:switch-project` to change context — don't manually edit PROJECTS.md

### Solo Developer with Reviewers

```
Developer: Plans and executes all phases
Reviewer: Reviews specs (via /dgs:refine-spec) and PRs

Config:
  branching_strategy: "phase"
  review: { openai: {...}, gemini: {...} }
```

This is the simplest multi-user pattern. The developer does all execution; others contribute through ideas, spec refinement, and code review on the phase branches.

---

## Quick Reference: Parallel Safety Matrix

| Activity | + Ideas/Specs | + Discuss Phase | + Plan Phase | + Execute Phase | + Quick Task |
|----------|:---:|:---:|:---:|:---:|:---:|
| **Ideas/Specs** | Yes | Yes | Yes | Yes | Yes |
| **Discuss Phase** | Yes | Yes | Yes | Yes | Yes |
| **Plan Phase** | Yes | Yes | Diff phase only | Diff phase only | Yes |
| **Execute Phase** | Yes | Yes | Diff phase only | Never same phase | Risky |
| **Quick Task** | Yes | Yes | Yes | Risky | Never |
| **Sync Pull** | Yes | Yes | Yes | Yes | Yes |
| **Sync Push** | Yes | Yes | Yes | Yes | Yes |

"Diff phase only" = safe if working on different phases. "Risky" = can work if touching different files/repos, but no guarantees. Sync operations are safe alongside any activity because they operate at the git level before/after DGS state modifications.

---

## Summary

DGS works well for multi-user teams when you follow three principles:

1. **Isolate execution** — One person per phase, always. Use phase branching to keep code separate.
2. **Collaborate on ideas and specs** — The idea-to-spec pipeline is naturally multi-user. Use it.
3. **Communicate on shared state** — Manifest counters, config changes, and merge order all need human coordination. DGS won't protect you from simultaneous updates.

The system's strength in multi-user scenarios comes from its file-per-entity design (one file per idea, one file per spec, one directory per phase) and its scoped commit messages. Its weakness is the shared state files (STATE.md, ROADMAP.md, config.json) that have no locking. Work with the grain of the system: parallelize where files are separate, serialize where state is shared.
