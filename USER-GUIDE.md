# DGS User Guide

A detailed reference for workflows, troubleshooting, and configuration. For quick-start setup, see the [README](../README.md).

---

## Sub-Guides

Detailed reference for specific topics lives in dedicated sub-documents:

- **[Command Reference](COMMAND-REFERENCE.md)** -- Every DGS command with flags, usage examples, and detailed behavior
- **[Configuration Guide](CONFIGURATION-GUIDE.md)** -- dgs.config.json schema, workflow toggles, cross-LLM review, conflict resolution
- **[Milestone Jobs & Execution Lifecycle](MILESTONE-JOBS-GUIDE.md)** -- Quick workflows, milestone jobs, phase audit, model profiles
- **[Setup Guide](SETUP-GUIDE.md)** -- Multi-repo setup, codebase mapping, product file structure
- **[Multi-User Approach Guide](MULTI_USER_APPROACH_GUIDE.md)** -- Coordinating DGS across multiple developers
- **[Context Monitor](context-monitor.md)** -- Context window usage monitoring and optimization

---

## Table of Contents

- [Workflow Diagrams](#workflow-diagrams)
- [Context Tiers](#context-tiers)
- [Usage Examples](#usage-examples)
- [Troubleshooting](#troubleshooting)
- [Recovery Quick Reference](#recovery-quick-reference)

---

## Workflow Diagrams

### Full Product Lifecycle

```
  ┌───────────────────────────────────────────────────────┐
  │                  PRODUCT SETUP                        │
  │  /dgs:init-product                                    │
  │  Creates .planning/, REPOS.md, PROJECTS.md            │
  │  /dgs:add-repo to register source repos               │
  └────────────────────────┬──────────────────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │         IDEAS & SPECS           │
          │                                 │
          │  /dgs:add-idea (capture)        │
          │  /dgs:write-spec (develop PRD)  │
          │  See: Ideas & Spec Development  │
          └────────────────┬────────────────┘
                           │
               Finalized spec ready
                    │            │
       ┌────────────▼──┐   ┌────▼──────────────┐
       │  New project?  │   │  New milestone?   │
       │                │   │                   │
       │ /dgs:new-      │   │ /dgs:new-         │
       │   project      │   │   milestone       │
       │ --auto @spec   │   │ --auto <spec-id>  │
       └───────┬────────┘   └─────┬─────────────┘
               │                  │
               └────────┬─────────┘
                        │
           ┌────────────▼─────────────┐
           │    PHASE LOOP            │
           │    (per milestone)       │
           │                          │
           │  discuss → plan →        │
           │  execute → verify        │
           │  See: Phase Lifecycle    │
           └────────────┬─────────────┘
                        │
           ┌────────────▼─────────────┐
           │  /dgs:audit-milestone    │
           │  /dgs:complete-milestone │
           └────────────┬─────────────┘
                        │
               Another milestone?
                  │          │
                 Yes         No
                  │          │
          ┌───────▼───┐  ┌──▼──────────────────┐
          │  Back to   │  │ /dgs:complete-      │
          │  Ideas &   │  │   project           │
          │  Specs     │  │ Start next project  │
          └────────────┘  └─────────────────────┘
```

### Ideas & Spec Development

```
  CAPTURE                    CURATE                     DEVELOP (optional)
  ─────────────────────      ──────────────────────     ──────────────────────
  /dgs:add-idea              /dgs:list-ideas            /dgs:discuss-idea
  /dgs:add-idea --auto       /dgs:update-idea           /dgs:research-idea
       │                     /dgs:reject-idea            /dgs:develop-idea
       ▼                     /dgs:restore-idea                │
  .planning/ideas/                │                           │
    pending/                      │                           │
       │                          │                           │
       │    Select 1+ pending ideas                           │
       │◄─────────────────────────┘                           │
       │                                                      │
       ├── Optionally develop idea(s) ───────────────────────►│
       │◄─────────────────────────────────────────────────────┘
       │
       ▼
  /dgs:write-spec
       │
       ├── Generate 9-section PRD
       ├── Auto-search for related content
       ├── Reads discussion + research context (if available)
       ├── User review (interactive mode)
       │
       ▼
  Cross-LLM Review Loop
       │
       ├── Send to OpenAI / Gemini reviewers
       ├── Accept, reject, or move feedback
       ├── Up to N rounds until convergence
       │
       ▼
  Save spec (status: draft)
  Source ideas move to done/
       │
       ▼
  REFINE & APPROVE
  ──────────────────────
  /dgs:refine-spec       (iterate on sections, version increments)
  /dgs:approve-spec      (validate and transition to final)
       │
       ├── Refine loop: edit → save → refine again
       ├── Approve: validates completeness, sets final
       │
       ▼
  Spec approved (status: final)
       │
       ├─────────────────────┐
       │                     │
       ▼                     ▼
  /dgs:new-project      /dgs:new-milestone
  --auto @spec.md       --auto <spec-id>
  (create project)      (add to existing project)
```

### Phase Lifecycle

```
  ┌──────────────────────────────────────────┐
  │           FOR EACH PHASE:                │
  │                                          │
  │  ┌────────────────────────────────┐      │
  │  │ /dgs:discuss-phase             │      │  <- Lock in preferences
  │  └──────────────┬─────────────────┘      │
  │                 │                        │
  │  ┌──────────────▼─────────────────┐      │
  │  │ /dgs:plan-phase                │      │  <- Research + Plan + Verify
  │  └──────────────┬─────────────────┘      │
  │                 │                        │
  │  ┌──────────────▼─────────────────┐      │
  │  │ /dgs:execute-phase             │      │  <- Parallel execution
  │  └──────────────┬─────────────────┘      │
  │                 │                        │
  │  ┌──────────────▼─────────────────┐      │
  │  │ /dgs:verify-work  OR           │      │  <- Manual UAT
  │  │ /dgs:audit-phase               │      │  <- Automated audit
  │  └──────────────┬─────────────────┘      │
  │                 │                        │
  │         Next Phase? ─────────────────────┘
  │                 │ No
  └─────────────────┼────────────────────────┘
                    │
                    ▼
            Milestone complete
```

### Planning Agent Coordination

```
  /dgs:plan-phase N
         │
         ├── Phase Researcher (x4 parallel)
         │     ├── Stack researcher
         │     ├── Features researcher
         │     ├── Architecture researcher
         │     └── Pitfalls researcher
         │           │
         │     ┌──────▼──────┐
         │     │ RESEARCH.md │
         │     └──────┬──────┘
         │            │
         │     ┌──────▼──────┐
         │     │   Planner   │  <- Reads PROJECT.md, REQUIREMENTS.md,
         │     │             │     CONTEXT.md, RESEARCH.md
         │     └──────┬──────┘
         │            │
         │     ┌──────▼───────────┐     ┌────────┐
         │     │   Plan Checker   │────>│ PASS?  │
         │     └──────────────────┘     └───┬────┘
         │                                  │
         │                             Yes  │  No
         │                              │   │   │
         │                              │   └───┘  (loop, up to 3x)
         │                              │
         │                        ┌─────▼──────┐
         │                        │ PLAN files │
         │                        └────────────┘
         └── Done
```

### Validation Architecture (Nyquist Layer)

During plan-phase research, DGS now maps automated test coverage to each phase
requirement before any code is written. This ensures that when Claude's executor
commits a task, a feedback mechanism already exists to verify it within seconds.

The researcher detects your existing test infrastructure, maps each requirement to
a specific test command, and identifies any test scaffolding that must be created
before implementation begins (Wave 0 tasks).

The plan-checker enforces this as an 8th verification dimension: plans where tasks
lack automated verify commands will not be approved.

**Output:** `{phase}-VALIDATION.md` -- the feedback contract for the phase.

**Disable:** Set `workflow.nyquist_validation: false` in `/dgs:settings` for
rapid prototyping phases where test infrastructure isn't the focus.

### Execution Wave Coordination

```
  /dgs:execute-phase N
         │
         ├── Analyze plan dependencies
         │
         ├── Wave 1 (independent plans):
         │     ├── Executor A (fresh 200K context) -> commit
         │     └── Executor B (fresh 200K context) -> commit
         │
         ├── Wave 2 (depends on Wave 1):
         │     └── Executor C (fresh 200K context) -> commit
         │
         └── Verifier
               └── Check codebase against phase goals
                     │
                     ├── PASS -> VERIFICATION.md (success)
                     └── FAIL -> Issues logged for /dgs:verify-work
```

### Brownfield Workflow (Existing Codebase)

```
  /dgs:map-codebase [<repo-name>]
         │
         ├── Per repo: 4 parallel mapper agents
         │   ├── Stack Mapper     -> codebase/<repo>/STACK.md, INTEGRATIONS.md
         │   ├── Arch Mapper      -> codebase/<repo>/ARCHITECTURE.md, STRUCTURE.md
         │   ├── Quality Mapper   -> codebase/<repo>/CONVENTIONS.md, TESTING.md
         │   └── Concern Mapper   -> codebase/<repo>/CONCERNS.md
         │
         ├── Synthesis: 7 unified top-level files
         │   └── codebase/ARCHITECTURE.md, STACK.md, STRUCTURE.md, ...
         │
         ├── Cross-repo analysis (2+ repos)
         │   └── codebase/CROSS-REPO.md
         │
         └── Secret scanning
                │
        ┌───────▼──────────┐
        │ /dgs:new-project │  <- Questions focus on what you're ADDING
        └──────────────────┘
```

---

## Context Tiers

DGS uses a 5-tier context loading system to control which project files each command reads. This prevents lightweight commands from wasting context budget on files they don't need, while ensuring planning and execution commands always have the information required for accurate work.

For the machine-readable tier definitions parsed at runtime, see `references/context-tiers.md` in the DGS installation directory.

### Tier Overview

| Tier | Name | Includes | Files Loaded |
|------|------|----------|--------------|
| 0 | none | -- | (no files) |
| 1 | lite | -- | PROJECT.md, STATE.md, config.json |
| 2 | planning | Tier 1 + | ROADMAP.md, REQUIREMENTS.md, REPOS.md, codebase/*.md |
| 3 | execution | Tier 2 + | Phase CONTEXT.md, RESEARCH.md, PLANs, SUMMARYs, milestone SUMMARYs |
| 4 | verification | Tier 3 + | VERIFICATION.md, UAT*.md |

### Tier 0: none (stateless)

| Command | Description |
|---------|-------------|
| `/dgs:help` | Show command reference |
| `/dgs:update` | Update DGS installation |
| `/dgs:set-profile <profile>` | Switch model profile |
| `/dgs:join-discord` | Join community |

### Tier 1: lite

| Command | Description |
|---------|-------------|
| `/dgs:add-doc <file>` | Attach supporting document |
| `/dgs:add-idea [--auto]` | Capture a new idea |
| `/dgs:add-todo [desc]` | Capture idea for later |
| `/dgs:add-repo [name]` | Register a sibling source repo |
| `/dgs:cancel-job <version>` | Cancel an in-progress job |
| `/dgs:capture-principle` | Capture design principle from context |
| `/dgs:check-todos [area]` | List pending todos |
| `/dgs:cleanup` | Archive completed milestone directories |
| `/dgs:complete-project` | Mark current project as completed |
| `/dgs:health [--repair]` | Validate .planning/ integrity |
| `/dgs:init-product` | Initialize product structure |
| `/dgs:list-docs` | List all supporting documents |
| `/dgs:list-ideas [--tag TAG]` | View ideas by state |
| `/dgs:list-jobs` | List milestone jobs by status |
| `/dgs:list-projects` | Show all projects with status |
| `/dgs:list-specs` | View specs by status |
| `/dgs:overlap-check` | Show repos touched by multiple projects |
| `/dgs:pause-work` | Create handoff when stopping |
| `/dgs:progress` | Check project status |
| `/dgs:reject-idea <id>` | Move idea to rejected |
| `/dgs:remove-doc [--move]` | Remove or move a document |
| `/dgs:remove-repo [name]` | Unregister a repo |
| `/dgs:restore-idea <id>` | Restore rejected/done idea to pending |
| `/dgs:rollback-job <version>` | Roll back code to pre-job state |
| `/dgs:resume-work` | Restore from last session |
| `/dgs:search <query>` | Search across ideas, specs, docs |
| `/dgs:settings` | Configure workflow agents |
| `/dgs:switch-project [name]` | Switch active project context |
| `/dgs:undo-consolidation [id]` | Undo a consolidation |
| `/dgs:update-idea <id>` | Edit idea or append note |

### Tier 2: planning

| Command | Description |
|---------|-------------|
| `/dgs:add-phase` | Append phase to roadmap |
| `/dgs:approve-spec <slug>` | Approve draft spec |
| `/dgs:consolidate-ideas [id...]` | Merge related ideas |
| `/dgs:create-milestone-job [version]` | Generate milestone build job |
| `/dgs:debug [desc]` | Systematic debugging with persistent state |
| `/dgs:develop-idea [id]` | Combined discussion then research |
| `/dgs:discuss-idea [id]` | Develop idea through discussion |
| `/dgs:discuss-phase [N]` | Capture implementation decisions |
| `/dgs:find-related-ideas [id]` | Find related ideas with scoring |
| `/dgs:import-spec <path>` | Import external document as spec |
| `/dgs:insert-phase [N]` | Insert urgent work between phases |
| `/dgs:list-phase-assumptions [N]` | See Claude's intended approach |
| `/dgs:map-codebase [repo]` | Map repos, synthesize codebase docs |
| `/dgs:new-milestone [name]` | Start next version cycle |
| `/dgs:new-project [--auto]` | Project identity: questioning → PROJECT.md |
| `/dgs:plan-milestone-gaps` | Create phases to close audit gaps |
| `/dgs:plan-phase [N]` | Create phase execution plans |
| `/dgs:quick [--full]` | Execute ad-hoc task with DGS guarantees |
| `/dgs:refine-spec <slug>` | Refine spec through conversation |
| `/dgs:remove-phase [N]` | Remove future phase, renumber |
| `/dgs:research-idea [id]` | Research idea feasibility |
| `/dgs:research-phase [N]` | Ecosystem research for complex domains |
| `/dgs:write-spec [id...]` | Draft PRD from ideas |

### Tier 3: execution

| Command | Description |
|---------|-------------|
| `/dgs:execute-phase <N>` | Execute plans in parallel waves |
| `/dgs:audit-phase <phase>` | Automated phase verification |
| `/dgs:run-job <version>` | Execute milestone build job |

### Tier 4: verification

| Command | Description |
|---------|-------------|
| `/dgs:audit-milestone` | Verify milestone completion |
| `/dgs:complete-milestone` | Archive milestone, tag release |
| `/dgs:validate-phase` | Validate phase plan structure |
| `/dgs:verify-phase` | Automated phase verification |
| `/dgs:verify-work [N]` | Interactive UAT |

### Scope Flags

Commands can load additional scoped files beyond their base tier using these flags:

- `--phase <N>` -- Loads phase-specific files (CONTEXT.md, RESEARCH.md, PLANs, SUMMARYs) for the given phase number. Used primarily by Tier 3 and Tier 4 commands.
- `--idea <id>` -- Loads the idea file and its `docs/` directory. Available at Tier 2 and above.
- `--spec <id>` -- Loads the spec file and its `docs/` directory. Available at Tier 2 and above.

For example, `dgs-tools context load-tier planning --idea 5` loads all Tier 2 files plus idea #5 and any documents in its `docs/` directory.

---

## Usage Examples

### New Project (Full Cycle)

```bash
claude --dangerously-skip-permissions
/dgs:new-project            # Answer questions, create PROJECT.md
/dgs:new-milestone          # Research, requirements, approve roadmap
/clear
/dgs:discuss-phase 1        # Lock in your preferences
/dgs:plan-phase 1           # Research + plan + verify
/dgs:execute-phase 1        # Parallel execution
/dgs:verify-work 1          # Manual UAT
/clear
/dgs:discuss-phase 2        # Repeat for each phase
...
/dgs:audit-milestone        # Check everything shipped
/dgs:complete-milestone     # Archive, tag, done
```

### New Project from Existing Document

```bash
/dgs:new-project --auto @prd.md   # Create PROJECT.md from your doc
/dgs:new-milestone               # Research, requirements, roadmap
/clear
/dgs:discuss-phase 1               # Normal flow from here
```

### Ideas to Project

```bash
/dgs:add-idea               # Capture ideas as they come
/dgs:add-idea               # Capture more over time
/dgs:list-ideas             # Review your backlog
/dgs:develop-idea           # Optional: discuss + research to refine
/dgs:write-spec             # Select ideas, draft PRD, cross-LLM review
/dgs:list-specs             # Verify spec is finalized
/dgs:new-project --auto @.planning/specs/my-feature.md   # Create project from spec
/dgs:new-milestone          # Research, requirements, roadmap
/clear
/dgs:discuss-phase 1        # Normal phase workflow from here
```

### Existing Codebase

```bash
/dgs:map-codebase           # Map all registered repos (per-repo docs + unified synthesis)
/dgs:new-project            # Questions focus on what you're ADDING
/dgs:new-milestone          # Research, requirements, roadmap
# (normal phase workflow from here)

# After making changes to a specific repo:
/dgs:map-codebase api-service   # Update just that repo's maps, regenerate unified files
```

### Quick Bug Fix

```bash
/dgs:quick
> "Fix the login button not responding on mobile Safari"
```

### Resuming After a Break

```bash
/dgs:progress               # See where you left off and what's next
# or
/dgs:resume-work            # Full context restoration from last session
```

### Preparing for Release

```bash
/dgs:audit-milestone        # Check requirements coverage, detect stubs
/dgs:plan-milestone-gaps    # If audit found gaps, create phases to close them
/dgs:complete-milestone     # Archive, tag, done
```

### Speed vs Quality Presets

| Scenario | Mode | Depth | Profile | Research | Plan Check | Verifier |
|----------|------|-------|---------|----------|------------|----------|
| Prototyping | `yolo` | `quick` | `budget` | off | off | off |
| Normal dev | `interactive` | `standard` | `balanced` | on | on | on |
| Production | `interactive` | `comprehensive` | `quality` | on | on | on |

### Mid-Milestone Scope Changes

```bash
/dgs:add-phase              # Append a new phase to the roadmap
# or
/dgs:insert-phase 3         # Insert urgent work between phases 3 and 4
# or
/dgs:remove-phase 7         # Descope phase 7 and renumber
```

---

## Troubleshooting

### "Project already initialized"

You ran `/dgs:new-project` but `.planning/PROJECT.md` already exists. This is a safety check. If you want to start over, delete the `.planning/` directory first.

### Context Degradation During Long Sessions

Clear your context window between major commands: `/clear` in Claude Code. DGS is designed around fresh contexts -- every subagent gets a clean 200K window. If quality is dropping in the main session, clear and use `/dgs:resume-work` or `/dgs:progress` to restore state.

### Plans Seem Wrong or Misaligned

Run `/dgs:discuss-phase [N]` before planning. Most plan quality issues come from Claude making assumptions that `CONTEXT.md` would have prevented. You can also run `/dgs:list-phase-assumptions [N]` to see what Claude intends to do before committing to a plan.

### Execution Fails or Produces Stubs

Check that the plan was not too ambitious. Plans should have 2-3 tasks maximum. If tasks are too large, they exceed what a single context window can produce reliably. Re-plan with smaller scope.

### Lost Track of Where You Are

Run `/dgs:progress`. It reads all state files and tells you exactly where you are and what to do next.

### Need to Change Something After Execution

Do not re-run `/dgs:execute-phase`. Use `/dgs:quick` for targeted fixes, or `/dgs:verify-work` to systematically identify and fix issues through UAT.

### Model Costs Too High

Switch to budget profile: `/dgs:set-profile budget`. Disable research and plan-check agents via `/dgs:settings` if the domain is familiar to you (or to Claude).

### Working on a Sensitive/Private Project

Set `commit_docs: false` during `/dgs:init-product` or via `/dgs:settings`. Add `.planning/` to your `.gitignore`. Planning artifacts stay local and never touch git.

### DGS Update Overwrote My Local Changes

Since v1.17, the installer backs up locally modified files to `dgs-local-patches/`. Run `/dgs:reapply-patches` to merge your changes back.

### Subagent Appears to Fail but Work Was Done

A known workaround exists for a Claude Code classification bug. DGS's orchestrators (execute-phase, quick) spot-check actual output before reporting failure. If you see a failure message but commits were made, check `git log` -- the work may have succeeded.

### Worktree Commands

Most users never run these directly — DGS manages worktrees automatically. These are for when things go wrong:

| Command | What it does |
|---------|-------------|
| `dgs-tools worktrees list` | Show all active worktrees (JSON output) |
| `dgs-tools worktrees create {slug} --type milestone` | Manually create a worktree |
| `dgs-tools worktrees remove {slug}` | Remove worktree, branch, and config entry |
| `dgs-tools worktrees setup {slug}` | Re-run setup command for a worktree |
| `dgs-tools worktrees prune` | Clean up orphaned entries (missing directories) |

See [How Git is Used](GIT-WORKFLOW.md) for a conceptual overview of the worktree model.

---

## Recovery Quick Reference

| Problem | Solution |
|---------|----------|
| Lost context / new session | `/dgs:resume-work` or `/dgs:progress` |
| Phase went wrong | `git revert` the phase commits, then re-plan |
| Need to change scope | `/dgs:add-phase`, `/dgs:insert-phase`, or `/dgs:remove-phase` |
| Milestone audit found gaps | `/dgs:plan-milestone-gaps` |
| Something broke | `/dgs:debug "description"` |
| Quick targeted fix | `/dgs:quick` |
| Plan doesn't match your vision | `/dgs:discuss-phase [N]` then re-plan |
| Costs running high | `/dgs:set-profile budget` and `/dgs:settings` to toggle agents off |
| Merge conflict during milestone | Automatic — DGS classifies and resolves; escalates LOW-confidence to you |
| Update broke local changes | `/dgs:reapply-patches` |
