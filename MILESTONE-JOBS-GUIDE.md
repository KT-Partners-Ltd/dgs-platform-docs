# Milestone Jobs & Execution Lifecycle

> See also: [USER-GUIDE.md](USER-GUIDE.md) for workflow overview and usage examples.

## Quick Workflows

Quick workflows handle work that doesn't need a full milestone: bug fixes, small improvements, debugging. DGS has two flavors based on context.

### Product-Level Quick

When there's no active milestone, or you use the `--main` flag:

- Creates an ephemeral worktree off main with a `quick/{title}` branch
- Full lifecycle: `dgs:complete-quick` to merge, `dgs:abandon-quick` to discard
- One active product-level quick at a time

### Milestone-Context Quick

When a milestone is active (and no `--main` flag):

- Runs inside the existing milestone worktree on the milestone branch
- No separate worktree or branch — changes merge with the milestone
- No `complete-quick` or `abandon-quick` needed

### dgs:quick

Creates a quick task. Auto-detects context:

```
/dgs:quick "fix auth token validation"          # milestone-context if milestone active
/dgs:quick --main "fix unrelated CSS bug"       # forces product-level (new worktree)
/dgs:quick --full "add input validation"         # tests and docs expected
/dgs:quick --debug "investigate flaky test"      # investigation focus
```

Flags `--full` and `--debug` change workflow guidance only. Git mechanics are identical across all flavors.

### dgs:complete-quick

Merges a product-level quick to main and cleans up. Only valid for product-level quicks (not milestone-context).

Steps performed automatically:

1. Pulls latest `base_branch` from remote
2. Rebases quick branch onto `base_branch` in worktree
3. If rebase conflicts — conflict-agent attempts auto-resolution, falls back to manual instructions
4. Fast-forward merges to `base_branch`
5. Pushes to remote
6. Removes quick worktree and branch

Output: `Quick 'fix-auth' merged to main (3 commits). Worktree cleaned up. Pushed to origin.`

### dgs:abandon-quick

Discards a product-level quick without merging. Removes worktree directory, git registration, and branch. Requires confirmation.

### When You're Mid-Quick and Something Urgent Arrives

- If trivial (1-10 lines): use `dgs:fast` — commits directly to main, no interference with your active quick
- If non-trivial: `complete-quick` or `abandon-quick` your current quick first, then start the new work
- You cannot have two product-level quicks simultaneously

### No Quick-to-Milestone Promotion

If a quick outgrows its scope, complete or abandon it and start a proper milestone. There is no way to convert a quick branch into a milestone branch.

---

## Milestone Jobs

Milestone jobs automate the discuss-plan-execute-verify cycle for an entire milestone. Instead of running commands phase by phase, create a job and let DGS execute the full pipeline with subagent isolation per step. Each step runs in a fresh context window, keeping quality consistent across long milestones.

### Creating a Milestone Job

```
/dgs:create-milestone-job [version] [--no-check]
```

Creates a milestone build job by reading your ROADMAP.md and generating a step sequence for every incomplete phase. When no version is given, the command auto-detects the active milestone.

The command:
1. Reads ROADMAP.md and determines each phase's state (unplanned, planned, or complete)
2. Generates a step sequence — skipping completed phases, adding discuss/plan/execute/verify for unplanned phases, and execute/verify for already-planned phases
3. Shows the full step list for your approval before writing
4. Writes the job file to `.planning/jobs/pending/milestone-{version}.md`

The `--no-check` flag omits the `audit-milestone` step from the generated job, useful when you want to run phases without the final audit cycle. Note: `complete-milestone` is never auto-run in jobs — it requires manual intervention (branch review, tag push, merge conflict resolution), so you always run it manually after the job completes.

**Examples:**
```
/dgs:create-milestone-job              # Auto-detect active milestone
/dgs:create-milestone-job v6 --no-check  # Skip audit/complete steps
```

### Running a Job

```
/dgs:run-job <version> [--dry-run]
```

Executes a milestone job end-to-end. Each step spawns an isolated subagent with fresh context — no context accumulation across steps. The job file is updated in real-time with status markers, timestamps, and error summaries.

On step failure, execution halts immediately and the error is recorded in the job file. On completion, the job moves from `in-progress/` to `completed/`.

### Resuming a Failed Job

Simply re-run `/dgs:run-job <version>` — completed steps are skipped automatically. Failed steps are reset to pending and re-executed. No special resume command is needed.

### Dry-Run Preview

```
/dgs:run-job <version> --dry-run
```

Shows what steps would execute without running them. Displays step markers to indicate status:

- `[x]` completed
- `[ ]` pending
- `[!]` failed
- `[>]` in-progress

### Audit-Gap-Fix Cycle

When `audit-milestone` finds gaps during a job, the orchestrator dynamically inserts plan + execute steps for new fix phases, then re-audits. This cycle repeats up to 3 times before halting with a clear message.

Gap-fix insertion is auto-approved -- no user prompt required. The 3-cycle limit is the safety net against runaway loops. Within the cycle, `plan-milestone-gaps` runs with `--auto`, and `plan-phase`/`execute-phase` steps use `--non-interactive` to keep the pipeline moving without human intervention.

Jobs created with `--no-check` skip the audit entirely — no audit step exists in the job file, so the gap-fix cycle never triggers.

Inserted steps appear in the job file before execution, making the job fully restartable even if interrupted mid-cycle.

### Listing and Cancelling Jobs

```
/dgs:list-jobs
```

Groups all jobs by status: **In Progress**, **Pending**, and **Completed**. Each job shows a progress fraction (completed/total steps).

```
/dgs:cancel-job <version>
```

Moves an in-progress job back to pending and resets any in-progress step markers. A job summary is generated after cancellation.

### Rolling Back a Job

```
/dgs:rollback-job <version>
```

Resets all code repos to the commit SHAs recorded when the job started. Planning artifacts (PLANs, CONTEXT.md, RESEARCH.md, etc.) are preserved -- only code repos and SUMMARY.md files are affected.

The rollback process:
1. Locates the job file and reads `StartShas` from the header
2. Previews which repos will be reset and which phases' SUMMARYs will be deleted
3. Requires explicit confirmation (type "rollback") since this is destructive
4. Runs `git reset --hard <sha>` in each code repo (NOT the planning repo)
5. Deletes SUMMARY.md files for phases that were executed by the job
6. Marks the job status as `rolled_back`

**Requirements:** The job must have been run with a version of DGS that records starting SHAs (v2.6+). Older jobs without `StartShas` in their header cannot be rolled back.

**Warning:** This permanently discards all code changes in the affected repos. Ensure you don't have uncommitted work in those repos before rolling back.

### Job File Format

Job files are Markdown documents with `[x]`/`[ ]`/`[!]` checkboxes:

- **Header:** Contains version, check flag, timestamps, and StartShas (JSON object mapping repo names to commit SHAs, recorded at job start)
- **Steps:** Each step is a checkbox line with the DGS command to execute
- **Timestamps:** Added alongside steps as they complete
- **Error summaries:** Appended for failed steps
- **Storage:** Files move between `.planning/jobs/pending/`, `in-progress/`, and `completed/` directories, which are auto-created when needed

### Job Summaries

Job summaries are auto-generated on completion, failure, and cancellation. They contain timing information, step results, errors, and details about any auto-resolve audit cycles. Summaries are written alongside the job file as `job-{version}-SUMMARY.md`.

### Command Reference

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:create-milestone-job [version] [--no-check]` | Generate build job from roadmap | Start automated milestone build |
| `/dgs:run-job <version>` | Execute job end-to-end | Run or resume a milestone job |
| `/dgs:run-job <version> --dry-run` | Preview steps without executing | Check what would run |
| `/dgs:list-jobs` | Show all jobs with status | See job pipeline |
| `/dgs:cancel-job <version>` | Cancel and reset job | Abort a running job |
| `/dgs:rollback-job <version>` | Roll back code to pre-job state | Undo a job's code changes |

---

## Phase Audit

Phase audit (`/dgs:audit-phase`) provides automated two-layer verification for completed phases.

### How It Works

```
/dgs:audit-phase <phase>
```

The pipeline runs five stages:

1. **Initialize** -- Parse phase argument, detect `--rerun-failed` flag, load repo context
2. **Test Execution (Layer 1)** -- Collect commands from VALIDATION.md and PLAN.md, execute with timeouts, parse results
3. **Structural Verification (Layer 2)** -- Spawn dgs-phase-verifier agent to cross-reference PLAN.md deliverables with actual files on disk (existence, substance, exports, must_haves, upstream wiring)
4. **Merge Results** -- Combine test results and structural gaps into UAT file, compute combined status
5. **Diagnose Gaps** -- If gaps found: spawn debug agents, create fix plans via planner/checker loop

**Status outcomes:**
- `passed` -- All tests pass, structural verification clean
- `passed` (with `human_needed`) -- Tests pass but some items need manual review
- `gaps_found` -> `diagnosed` -- Failures found, diagnosis complete, fix plans created

### Rerunning Failed Items

After gap closure, re-verify only previously-failed tests and structural checks:

```
/dgs:audit-phase <phase> --rerun-failed
```

This reads the existing UAT file, re-executes only failed test commands and re-checks only failed structural verifications, then merges results.

### Verification Decision Matrix

| Scenario | Command | Why |
|----------|---------|-----|
| Manually test features | `/dgs:verify-work` | Interactive UAT with human judgment |
| Running milestone job (automated) | `/dgs:audit-phase` | Programmatic verification in pipeline |
| Quick automated check after execute-phase | `/dgs:audit-phase` | Runs tests + structural verification |
| After gap closure, re-check failed items | `/dgs:audit-phase --rerun-failed` | Re-verify specific failures only |
| All phases done, check cross-phase integration | `/dgs:audit-milestone` | Milestone-level completeness |
| Rubber-stamp verification (skip it) | `/dgs:verify-work --auto` | Auto-pass without testing |

### Phase Gap-Fix Cycles

When audit-phase runs inside a milestone job and produces `diagnosed` status, the job orchestrator automatically inserts a phase-level gap-fix cycle:

1. `execute-phase <N> --gaps-only` -- Execute the fix plans
2. `audit-phase <N> --rerun-failed` -- Re-verify only failed items

Maximum 2 gap-fix cycles per phase. The phase-level `PhaseFixCycle` counter is independent from the milestone-level `GapFixCycle` budget. If the limit is exceeded, the job escalates to milestone-level handling.

### Integration with Milestone Jobs

When you create a milestone job with `/dgs:create-milestone-job`, the generated step sequence automatically includes:
- `map-codebase` before each `plan-phase` step (keeps codebase documentation current)
- `audit-phase` after each `execute-phase` step (automated verification)

Job summaries report any outstanding human-needed verifications that remain after job completion.

### Artifacts

| File | Location | Contents |
|------|----------|----------|
| UAT file | `{phase}-UAT.md` in phase directory | Test results, structural gaps, severity |
| Log file | `{phase}-UAT-LOG.md` in phase directory | Full stdout/stderr per command |

---

### Model Profiles (Per-Agent Breakdown)

| Agent | `quality` | `balanced` | `budget` |
|-------|-----------|------------|----------|
| dgs-planner | Opus | Opus | Sonnet |
| dgs-roadmapper | Opus | Sonnet | Sonnet |
| dgs-executor | Opus | Sonnet | Sonnet |
| dgs-phase-researcher | Opus | Sonnet | Haiku |
| dgs-project-researcher | Opus | Sonnet | Haiku |
| dgs-research-synthesizer | Sonnet | Sonnet | Haiku |
| dgs-debugger | Opus | Sonnet | Sonnet |
| dgs-codebase-mapper | Sonnet | Haiku | Haiku |
| dgs-phase-verifier | Sonnet | Sonnet | Haiku |
| dgs-verifier | Sonnet | Sonnet | Haiku |
| dgs-plan-checker | Sonnet | Sonnet | Haiku |
| dgs-integration-checker | Sonnet | Sonnet | Haiku |

**Profile philosophy:**
- **quality** -- Opus for all decision-making agents, Sonnet for read-only verification. Use when quota is available and the work is critical.
- **balanced** -- Opus only for planning (where architecture decisions happen), Sonnet for everything else. The default for good reason.
- **budget** -- Sonnet for anything that writes code, Haiku for research and verification. Use for high-volume work or less critical phases.
