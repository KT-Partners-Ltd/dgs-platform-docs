# DGS User Guide

A detailed reference for workflows, troubleshooting, and configuration. For quick-start setup, see the [README](../README.md).

---

## Table of Contents

- [Workflow Diagrams](#workflow-diagrams)
- [Multi-Repo Setup](#multi-repo-setup)
- [Codebase Mapping](#codebase-mapping)
- [Context Tiers](#context-tiers)
- [Command Reference](#command-reference)
  - [Command Hierarchy](#command-hierarchy)
- [Configuration Reference](#configuration-reference)
- [Conflict Resolution](#conflict-resolution)
- [Milestone Jobs](#milestone-jobs)
- [Phase Audit](#phase-audit)
- [Usage Examples](#usage-examples)
- [Troubleshooting](#troubleshooting)
- [Recovery Quick Reference](#recovery-quick-reference)
- [Product File Structure](#product-file-structure)

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

## Multi-Repo Setup

DGS supports managing multiple source code repos from a single planning repo. All repos sit as siblings in the same parent directory:

```
~/projects/
├── my-product/              <- planning repo (run DGS commands here)
│   └── .planning/
│       ├── REPOS.md         <- repo registry
│       ├── PROJECTS.md
│       ├── config.json
│       └── phases/
├── api-service/             <- source repo (sibling)
├── web-app/                 <- source repo (sibling)
└── mobile-app/              <- source repo (sibling)
```

**Setting up:**

```bash
cd ~/projects
mkdir my-product && cd my-product
/dgs:init-product                    # creates .planning/ structure
/dgs:add-repo api-service            # registers ../api-service
/dgs:add-repo web-app                # registers ../web-app
/dgs:add-repo mobile-app             # registers ../mobile-app
/dgs:new-project                     # start your first project
```

**How it works:**
- All DGS commands run from the planning repo (`my-product/`)
- Repos are registered in `.planning/REPOS.md` with `../repo-name` paths
- `/dgs:add-repo` defaults to `../` paths and validates the sibling directory exists
- Plans reference repos by name in `<repos>` tags; file paths are repo-relative
- Execution resolves paths via REPOS.md lookup at runtime

**REPOS.md format:**

```markdown
| Name | Path | GitHub URL | Description |
|------|------|------------|-------------|
| api-service | ../api-service | https://github.com/org/api | Node.js API |
| web-app | ../web-app | https://github.com/org/web | React frontend |
```

**Migrating from nested layout:**

If you previously used DGS with repos nested inside the planning repo:

1. Move repos to be siblings:
   ```bash
   cd ~/projects/my-product
   mv api-service ../api-service
   mv web-app ../web-app
   ```
2. Update REPOS.md paths from `./api-service` to `../api-service`
3. Remove the DGS-managed section from `.gitignore` (no longer needed)
4. Run `/dgs:health` to verify all repos are reachable

---

## Codebase Mapping

`/dgs:map-codebase` produces structured documentation of your codebase. In multi-repo setups, it creates per-repo subdirectories plus unified top-level files that existing workflows consume without modification.

### Directory Structure

After running `/dgs:map-codebase` with two registered repos (`api-service` and `web-app`):

```
.planning/codebase/
├── api-service/                # Per-repo maps
│   ├── STACK.md
│   ├── ARCHITECTURE.md
│   ├── STRUCTURE.md
│   ├── CONVENTIONS.md
│   ├── TESTING.md
│   ├── INTEGRATIONS.md
│   └── CONCERNS.md
├── web-app/                    # Per-repo maps
│   ├── STACK.md
│   ├── ARCHITECTURE.md
│   ├── STRUCTURE.md
│   ├── CONVENTIONS.md
│   ├── TESTING.md
│   ├── INTEGRATIONS.md
│   └── CONCERNS.md
├── ARCHITECTURE.md             # Unified (labeled repo sections + cross-repo summary)
├── STACK.md                    # Unified
├── STRUCTURE.md                # Unified
└── CROSS-REPO.md               # Shared deps, API boundaries, patterns (2+ repos)
```

**Per-repo documents** contain detailed analysis of each repo's codebase by 4 parallel mapper agents:
- **Stack Mapper**: STACK.md (dependencies, runtime) and INTEGRATIONS.md (APIs, external services)
- **Architecture Mapper**: ARCHITECTURE.md (layers, patterns) and STRUCTURE.md (directory layout)
- **Quality Mapper**: CONVENTIONS.md (code style, patterns) and TESTING.md (test approach, coverage)
- **Concerns Mapper**: CONCERNS.md (tech debt, risks, known issues)

**Unified top-level files** are synthesized from per-repo maps. They contain labeled repo sections plus a cross-repo summary, and are what existing workflows (write-spec, refine-spec, import-spec, new-project) consume as context.

**CROSS-REPO.md** (generated for 2+ repos) contains comparison tables:
- Shared dependencies across repos
- API boundaries (provider/consumer relationships)
- Common patterns (architecture, testing, conventions that match)
- Divergent patterns (where repos differ, documented neutrally)

### Refresh vs Update Mode

| Mode | Trigger | What happens |
|------|---------|--------------|
| **Refresh** | `/dgs:map-codebase` (no args) | Deletes all codebase docs, remaps every registered repo, regenerates unified files and CROSS-REPO.md |
| **Update** | `/dgs:map-codebase <repo-name>` | Deletes only that repo's subdirectory, remaps it, regenerates unified files from all repos with content |

Update mode is useful after making changes to a specific repo -- you get fresh maps for that repo without re-mapping everything.

The `--only <repo-name>` flag works identically to the positional argument and is kept for backward compatibility.

### Secret Scanning

All generated codebase files (per-repo and unified) are scanned for accidentally included secrets. Patterns detected include API keys, connection strings (PostgreSQL, MongoDB, MySQL, Redis, AMQP), hardcoded passwords (8+ characters), and common credential patterns. Matches produce warnings in the mapping summary.

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
| `/dgs:new-project [--auto]` | Full project initialization |
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

## Command Reference

### Command Hierarchy

DGS commands span a range of scope and automation. The hierarchy below orders them from lightest to heaviest — choose the smallest command that fits your task.

| Tier | Command | Scope | Subagents | Best For |
|------|---------|-------|-----------|----------|
| 1 | `/dgs:fast <desc>` | 1-3 files, ~30 lines | None | Typo fixes, config tweaks, one-line changes |
| 2 | `/dgs:quick` | Small feature or bug fix | Planner + executor | Bug fixes, small features, ad-hoc tasks |
| 3 | `/dgs:quick --full` | Medium task needing quality guarantees | Planner + checker + executor + verifier | Tasks where you want plan-checking and post-execution verification without full milestone ceremony |
| 4 | `/dgs:execute-phase <N>` | Full planned phase | Research + planner + checker + executor + verifier | Milestone work with research, planning, and verification |
| 5 | `/dgs:debug [desc]` | Open-ended investigation | Debug agents | Systematic diagnosis when something is broken |

**How to choose:** Start with `/dgs:fast` for trivial edits. If the scope warning fires (more than 3 files or 30 lines), step up to `/dgs:quick`. Use `/dgs:quick --full` when you want verification but don't need milestone ceremony. Reserve `/dgs:execute-phase` for planned work with requirements and roadmap tracking.

**What `--full` adds:** The `--full` flag on `/dgs:quick` enables the plan-checker agent (validates plans achieve the task goal, up to 2 iterations) and the post-execution verifier agent (confirms deliverables match intent). These are the same quality gates used in `/dgs:execute-phase`, applied to an ad-hoc task.

### Core Workflow

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:new-project` | Full project init: questions, research, requirements, roadmap | Start of a new project |
| `/dgs:discuss-phase [N]` | Capture implementation decisions | Before planning, to shape how it gets built |
| `/dgs:plan-phase [N]` | Research + plan + verify | Before executing a phase |
| `/dgs:execute-phase <N>` | Execute all plans in parallel waves | After planning is complete |
| `/dgs:verify-work [N] [--auto]` | Interactive UAT (`--auto` for rubber-stamp pass) | After execution completes |
| `/dgs:audit-phase <phase> [--rerun-failed]` | Automated phase verification (tests + structural inspection) | After execution completes |
| `/dgs:audit-milestone` | Verify milestone met its definition of done | Before completing milestone |
| `/dgs:complete-milestone` | Archive milestone, tag release | All phases verified |
| `/dgs:new-milestone [name]` | Start next version cycle | After completing a milestone |

#### Core Workflow Details

**`/dgs:new-project`**
Initialize a new project through questioning, research, requirements, and roadmap creation.

- `--auto @file.md`: Automated init from a PRD or idea document
- `--auto <spec-id>`: Automated init from a finalized spec

Usage: `/dgs:new-project` (interactive questioning flow)
Usage: `/dgs:new-project --auto @prd.md` (from idea document)
Usage: `/dgs:new-project --auto spec-review-config` (from finalized spec)

**`/dgs:discuss-phase [N]`**
Extract implementation decisions that guide research and planning.

- `--auto`: Auto-advance through discussion into plan-phase and execute-phase

Usage: `/dgs:discuss-phase 3` (discuss phase 3 interactively)
Usage: `/dgs:discuss-phase 3 --auto` (auto-advance pipeline)

**`/dgs:plan-phase [N]`**
Create executable plans for a phase with integrated research and verification.

- `--research`: Force re-run research even if RESEARCH.md exists
- `--skip-research`: Skip research step entirely (domain is familiar)
- `--gaps`: Plan gap-closure phases from VERIFICATION.md
- `--skip-verify`: Skip plan verification step
- `--non-interactive`: Skip interactive prompts (auto-resolve confirmation gates) without auto-advancing to execute-phase
- `--auto`: Auto-advance through plan-phase into execute-phase (implies `--non-interactive`)

Usage: `/dgs:plan-phase 3` (full research + plan + verify)
Usage: `/dgs:plan-phase 3 --skip-research` (plan without research)
Usage: `/dgs:plan-phase 3 --gaps` (plan fixes for verification gaps)
Usage: `/dgs:plan-phase 3 --non-interactive` (plan without prompts, stop after planning)

**`/dgs:execute-phase <N>`**
Execute all plans in a phase using wave-based parallel execution.

- `--non-interactive`: Auto-approve checkpoints and verification without auto-advancing to next phase
- `--auto`: Auto-advance to next phase after completion (implies `--non-interactive`)
- `--gaps-only`: Execute only gap-closure plans

Usage: `/dgs:execute-phase 3` (execute phase 3)
Usage: `/dgs:execute-phase 3 --non-interactive` (auto-approve without auto-advancing)
Usage: `/dgs:execute-phase 3 --auto` (hands-off execution pipeline)
Usage: `/dgs:execute-phase 3 --gaps-only` (execute only gap-closure plans)

**`/dgs:new-milestone [name]`**
Start a new milestone cycle for an existing project.

- `--auto <spec-id>`: Derive milestone from a finalized spec without interactive questioning

Usage: `/dgs:new-milestone` (interactive milestone creation)
Usage: `/dgs:new-milestone --auto spec-review-config` (from finalized spec)

**`/dgs:audit-phase <phase> [--rerun-failed]`**
Automated phase-level verification combining test execution with structural inspection.

- Collects test commands from VALIDATION.md and PLAN.md (`<verify><automated>` blocks) with deduplication
- Runs the full test suite first, then per-task verify commands (skipping covered duplicates)
- Each command has a 120-second timeout; infrastructure vs code failures classified separately
- False-positive exit code 0 results caught by output sanity checks
- Spawns the dgs-phase-verifier agent to cross-reference PLAN.md deliverables with actual files (existence, substance, exports, must_haves, upstream wiring)
- Structural gaps appear in the UAT with `source: structural_verification` and `gap_type: structural`
- When failures are found, diagnosis pipeline runs automatically: debug agents investigate, fix plans created
- Manual-only tests flagged as `human_needed` without blocking the pipeline
- Full raw output saved to a log file alongside the UAT file

Usage: `/dgs:audit-phase 41` (full audit of phase 41)
Usage: `/dgs:audit-phase 41 --rerun-failed` (re-verify only previously-failed items)

**`/dgs:verify-work` flag notes:**
- `--auto` is the only remaining flag (rubber-stamp pass for jobs)
- Without flags, verify-work runs the interactive human UAT flow

**`/dgs:complete-milestone <version>`**
Archive completed milestone and prepare for next version.

- Creates MILESTONES.md entry with stats
- Archives full details to milestones/ directory
- Creates git tag for the release
- Prepares workspace for next version

**Job-mode behavior:** When invoked from a milestone job with `<job-mode>silent</job-mode>`, all interactive gates are auto-resolved: scope confirmation is auto-approved, incomplete requirements are acknowledged, phase archival is skipped, branch handling is kept for manual review, and tag push is skipped for safety.

Usage: `/dgs:complete-milestone 1.0.0`

### Navigation

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:progress` | Show status and next steps | Anytime -- "where am I?" |
| `/dgs:resume-work` | Restore full context from last session | Starting a new session |
| `/dgs:pause-work` | Save context handoff | Stopping mid-phase |
| `/dgs:help` | Show all commands | Quick reference |
| `/dgs:update` | Update DGS with changelog preview | Check for new versions |
| `/dgs:join-discord` | Open Discord community invite | Questions or community |

### Phase Management

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:add-phase` | Append new phase to roadmap | Scope grows after initial planning |
| `/dgs:insert-phase [N]` | Insert urgent work (decimal numbering) | Urgent fix mid-milestone |
| `/dgs:remove-phase [N]` | Remove future phase and renumber | Descoping a feature |
| `/dgs:list-phase-assumptions [N]` | Preview Claude's intended approach | Before planning, to validate direction |
| `/dgs:plan-milestone-gaps` | Create phases for audit gaps | After audit finds missing items |
| `/dgs:research-phase [N]` | Deep ecosystem research only | Complex or unfamiliar domain |

#### Phase Management Details

**`/dgs:plan-milestone-gaps`**
Create phases to close gaps identified by milestone audit.

- Reads MILESTONE-AUDIT.md and groups gaps into phases
- Prioritizes by requirement priority (must/should/nice)
- Adds gap closure phases to ROADMAP.md
- `--auto`: Non-interactive mode -- auto-approve gap closure phases without user confirmation

Usage: `/dgs:plan-milestone-gaps` (interactive -- confirms before creating phases)
Usage: `/dgs:plan-milestone-gaps --auto` (non-interactive gap closure)

### Brownfield & Utilities

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:map-codebase [<repo-name>]` | Map repos with per-repo + unified docs | Before `/dgs:new-project`, after repo changes |
| `/dgs:fast <desc>` | Trivial edit with single atomic commit, no subagents | Typo fix, config tweak, one-line change |
| `/dgs:quick` | Ad-hoc task with DGS guarantees | Bug fixes, small features, config changes |
| `/dgs:debug [desc]` | Systematic debugging with persistent state | When something breaks |
| `/dgs:add-todo [desc]` | Capture an idea for later | Think of something during a session |
| `/dgs:check-todos [area]` | List pending todos, optionally filtered by area | Review captured ideas |
| `/dgs:cleanup` | Archive accumulated phase directories | After completing milestones, reduce clutter |
| `/dgs:settings` | Configure workflow toggles and model profile | Change model, toggle agents |
| `/dgs:set-profile <profile>` | Quick profile switch (`quality`, `balanced`, `budget`) | Change cost/quality tradeoff |
| `/dgs:reapply-patches` | Restore local modifications after update | After `/dgs:update` if you had local edits |

#### Brownfield & Utilities Details

**`/dgs:map-codebase [<repo-name>]`**
Map registered repos to produce structured codebase documentation with per-repo detail and unified synthesis.

- **Refresh mode** (no args): Clears all codebase maps and remaps every registered repo from scratch
- **Update mode** (`<repo-name>` or `--only <name>`): Re-maps only the specified repo, then regenerates unified files from all repos with content
- Spawns 4 parallel mapper agents per repo (Stack, Architecture, Quality, Concerns)
- Creates 7 documents per repo in `codebase/<repo-name>/`
- Synthesizes unified top-level files (ARCHITECTURE.md, STACK.md, STRUCTURE.md) from per-repo maps
- Generates CROSS-REPO.md with comparison tables: shared dependencies, API boundaries, common and divergent patterns (2+ repos only)
- Runs secret scanning across all generated files
- Validates repo name against REPOS.md when targeting a specific repo

Usage: `/dgs:map-codebase` (refresh all repos)
Usage: `/dgs:map-codebase api-service` (update specific repo only)

**`/dgs:quick`**
Execute small, ad-hoc tasks with atomic commits and STATE.md tracking.

- `--full`: Enable plan-checking and post-execution verification for quality guarantees
- `--fast`: Equivalent to `/dgs:fast` — no subagents, single atomic commit

Usage: `/dgs:quick` (prompts for task description)
Usage: `/dgs:quick fix the login button` (with inline description)
Usage: `/dgs:quick --full add input validation` (with plan-checking and verification)

**`/dgs:fast <description>`**
Make a trivial edit with a single atomic commit. The lightest DGS command.

- No subagents — orchestrator makes edits directly
- Infers conventional commit prefix (fix:/feat:/chore:/docs:/refactor:)
- Shares quick task ID counter (no ID collisions with `/dgs:quick`)
- Warns if scope exceeds 3 files or 30 lines (suggests `/dgs:quick` instead)
- `--dry-run`: Show proposed changes as diff without modifying files

Usage: `/dgs:fast fix the login button color` (make edit and commit)
Usage: `/dgs:fast update timeout --dry-run` (preview changes first)

### Ideas & Specs

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:add-idea` | Capture a new idea | When inspiration strikes |
| `/dgs:list-ideas` | View ideas by state | Review your idea backlog |
| `/dgs:discuss-idea [id]` | Develop idea through structured discussion | Refine problem and approach before researching |
| `/dgs:research-idea [id]` | Research idea's feasibility and landscape | Investigate technical options before spec writing |
| `/dgs:develop-idea [id]` | Discussion then research in one flow | Full idea development in one session |
| `/dgs:update-idea <id>` | Edit or replace idea content | Refine an idea over time |
| `/dgs:reject-idea <id>` | Move idea to rejected | Idea is out of scope |
| `/dgs:restore-idea <id>` | Restore to pending | Reconsider a rejected/done idea |
| `/dgs:consolidate-ideas [id...] [--title "..."]` | Merge related ideas into one AI-synthesised idea | Reduce idea backlog by merging overlapping ideas |
| `/dgs:find-related-ideas [id] [--threshold ...]` | Find ideas related to a selected idea | Discover overlapping ideas before consolidation |
| `/dgs:undo-consolidation [id]` | Undo a consolidation | Reverse a consolidation that did not turn out right |
| `/dgs:write-spec` | Draft PRD from ideas with cross-LLM review | Ready to formalize ideas into a spec |
| `/dgs:import-spec <path>` | Import external document as DGS spec | Have an external document to bring into DGS |
| `/dgs:list-specs` | View specs by status | Check spec pipeline |
| `/dgs:refine-spec <slug> [--section <N>]` | Refine spec through conversational editing | Iterate on a spec before approval |
| `/dgs:approve-spec <slug>` | Approve draft spec after validation | Finalize a spec for implementation |

#### Ideas & Specs Details

**`/dgs:add-idea`**
Capture a new idea into the ideas system interactively or from conversation context.

- `--auto`: Auto-extract idea title, body, and tags from conversation context without prompts

Usage: `/dgs:add-idea` (interactive: prompts for title, body, tags)
Usage: `/dgs:add-idea --auto` (extracts from conversation context)

**`/dgs:list-ideas`**
View ideas grouped by state (Pending, Done, Rejected) with development status indicators.

- `--tag TAG`: Filter ideas by tag
- `--pending`: Show only pending ideas
- `--done`: Show only completed ideas
- `--rejected`: Show only rejected ideas

Usage: `/dgs:list-ideas` (show all ideas)
Usage: `/dgs:list-ideas --pending` (focus on actionable ideas)
Usage: `/dgs:list-ideas --tag api` (filter by tag)

**`/dgs:discuss-idea [id]`**
Develop and refine an idea through structured, inline conversation.

- Walks through three phases: **Understanding** (Claude restates the idea, asks clarifying questions), **Exploration** (raises alternatives, conflicts, scope concerns), and **Refinement** (proposes refined problem, approach, and open questions)
- Appends a Discussion Log entry to the idea file with: Key Insights, Refined Problem, Refined Approach, Open Questions, and Decision
- When prior discussion exists, builds on it -- jumps to unresolved Open Questions rather than starting fresh
- User can exit mid-discussion; partial progress is always saved on exit (no prompt)
- Discussion results are committed: `docs: discuss idea #N — title`

Usage: `/dgs:discuss-idea 5` (discuss idea #5)
Usage: `/dgs:discuss-idea` (prompts to select from pending ideas)

**`/dgs:research-idea [id]`**
Research an idea's feasibility and technical landscape via subagent.

- Investigates five adaptive dimensions: web search, codebase analysis, landscape survey, approach identification, and feasibility assessment
- For multi-repo projects, research is partitioned by repo with separate sections
- Creates a structured research document at `.planning/docs/ideas/pending/{slug}-research.md` with frontmatter (type, idea_id, idea_title, date, repos_analysed)
- Appends a Research Log entry to the idea file with: Summary, Document link, Key Finding, and Recommendation
- Can be run multiple times -- each run appends a new Research Log entry and updates the research document
- Both the research document and updated idea file are committed together

Usage: `/dgs:research-idea 5` (research idea #5)
Usage: `/dgs:research-idea` (prompts to select from pending ideas)

**`/dgs:develop-idea [id]`**
Develop an idea through discussion then research as a single continuous flow.

- Runs discussion first (Understanding, Exploration, Refinement), then research (five adaptive dimensions)
- Discussion is committed before research begins, so research reads the freshly-updated idea
- If discussion concludes the idea is not viable, research is skipped entirely
- If user exits mid-discussion, partial progress is saved but research does not run
- When re-running on a previously-developed idea, offers a choice: re-do both, just discuss, or just research
- Combines the output of both commands: Discussion Log entry, research document, and Research Log entry

Usage: `/dgs:develop-idea 5` (develop idea #5)
Usage: `/dgs:develop-idea` (prompts to select from pending ideas)

**`/dgs:update-idea <id>`**
Edit or replace idea content, or append a timestamped note.

- `--note "text"`: Append a timestamped note without rewriting the idea body

Usage: `/dgs:update-idea 5` (edit idea #5)
Usage: `/dgs:update-idea 5 --note "discussed with team, they like it"` (append note)

**`/dgs:write-spec [id...]`**
Draft a structured PRD spec from pending ideas, send through cross-LLM review, and finalize.

- `--auto`: Skip interactive prompts (requires idea IDs as arguments)

Usage: `/dgs:write-spec` (interactive: select ideas, review draft, approve)
Usage: `/dgs:write-spec 1 3` (pre-select ideas 1 and 3)
Usage: `/dgs:write-spec 1 3 --auto` (fully automated spec creation)

**`/dgs:list-specs`**
View specs grouped by status (Draft, Final) with version and implementation tracking.

- `--draft`: Show only draft specs
- `--final`: Show only finalized specs
- Output includes Version column (spec version number) and Implementation column (linked milestone status: none, in-progress, or completed)

Usage: `/dgs:list-specs` (show all specs)
Usage: `/dgs:list-specs --final` (see specs ready for projects)

**`/dgs:import-spec <path> [--ideas <id...>]`**
Import an external document and convert it into a structured 9-section PRD spec using AI-powered restructuring.

Supported file types: PDF, markdown, and images.

- `--ideas <id...>`: Link the imported spec to one or more existing idea IDs

**How it works:**

1. Run the command with a path to your external document. DGS reads the file and converts its content into the standard 9-section PRD structure (Problem Statement, Goals, Non-Goals, User Stories, Requirements, Success Metrics, Open Questions, Timeline Considerations, Implementation Notes). Every detail from the source document is preserved — content is restructured, not summarized.

2. DGS presents the converted spec for your review. You have four options:
   - **Save** — accept the conversion and save it as a draft spec. The original document is preserved as an attachment in the spec's docs directory.
   - **Edit** — provide feedback on specific sections. DGS revises the conversion based on your input and presents the updated version for another review.
   - **Discard** — cancel the import without saving anything.
   - **Restart** — regenerate the conversion from scratch.

3. When you save, DGS creates the spec with `draft` status, copies the original document into the spec's attachment directory, and commits everything in a single atomic git commit.

If your project has codebase maps (`.planning/codebase/`) or product docs (`.planning/docs/product/`), DGS loads them as context during conversion. This means the Implementation Notes section references real modules and patterns from your codebase rather than generic placeholders.

Usage: `/dgs:import-spec ./mobile-app-redesign.pdf`
Usage: `/dgs:import-spec ./api-overhaul.md --ideas 3 7`

#### Spec Lifecycle

Specs follow a two-state machine: **draft** and **final**. A spec starts as draft when created via `/dgs:write-spec` or `/dgs:import-spec`. It can be iteratively refined (each refinement increments the version) and formally approved to reach final status. All lifecycle changes are tracked in the spec's Refinement Log.

**`/dgs:refine-spec <slug> [--section <N>]`**
Refine a spec through a conversational editing session with Claude as a collaborative thinking partner.

- Opens the spec for discussion: Claude presents the current content, and you iterate on improvements together
- `--section <N>`: Focus refinement on a specific section by number or heading name (case-insensitive substring match)
- All changes are written atomically at the end of the session -- no partial writes during the conversation
- Version increments by 0.1 on each refinement (e.g., 1.0 -> 1.1 -> 1.2)
- A Refinement Log entry is appended recording the date, version, and a summary of changes
- When refining a final-status spec, you are warned that the spec will be moved back to draft. You can proceed or cancel

Usage: `/dgs:refine-spec spec-lifecycle-management` (refine entire spec)
Usage: `/dgs:refine-spec spec-lifecycle-management --section 5` (focus on section 5)
Usage: `/dgs:refine-spec spec-lifecycle-management --section "open questions"` (focus by heading name)

**`/dgs:approve-spec <slug>`**
Approve a draft spec after completeness validation, transitioning it to final status.

- Validates required sections are present (Problem Statement, Goals, Requirements)
- Checks that at least one P0 requirement exists in the Requirements section
- Checks for blocking open questions and flags them as errors
- Missing optional sections (Success Metrics, Implementation Notes, User Stories) produce warnings -- you can confirm to proceed or cancel
- On approval: status transitions to final, approved_date is set, and a Refinement Log entry records the approval
- Running on an already-final spec shows an informational message and makes no changes

Usage: `/dgs:approve-spec spec-lifecycle-management` (approve the spec)

#### Idea Consolidation Details

**`/dgs:consolidate-ideas [id...] [--title "..."]`**
Merge two or more related pending ideas into a single AI-synthesised idea with full lineage tracking and history preservation.

- Without IDs: lists pending ideas for interactive selection (minimum 2 required)
- AI synthesises a coherent body that captures the combined intent of all source ideas -- not a mechanical concatenation
- Tags are the deduplicated union of all source idea tags, sorted alphabetically
- Discussion logs from all sources are merged with `[from #ID]` attribution, ordered chronologically
- Research logs from all sources are merged with `[from #ID]` attribution, ordered chronologically
- Notes from all sources are merged with source attribution for reference
- A provenance line is appended to the new idea body listing all source IDs and titles
- The user sees a diff-style preview: source ideas with title, tags, and body excerpt alongside the synthesised result
- User can approve, request edits to the synthesised content, or cancel
- After approval, source ideas move to `consolidated/` state with `consolidated_into` references pointing to the new idea
- The new idea is created in `pending/` with `consolidated_from` listing all source IDs
- Everything is committed atomically: `ideas: consolidate #id1, #id2 -> #new-id - title`
- `--title "..."`: Provide an explicit title instead of having Claude generate one

**How it works:**

1. Run the command with idea IDs or select interactively. DGS validates all source ideas exist and are in pending state.
2. DGS reads the full content of each source idea (body, notes, discussion logs, research logs) and generates a coherent synthesised body.
3. A diff-style preview shows what will happen: your N source ideas alongside the proposed merged idea. Review the synthesised body, title, and merged tags.
4. Choose to approve (creates the consolidated idea), edit (describe changes and review again), or cancel (no changes).
5. On approval, the CLI creates the new idea, moves sources to `consolidated/`, and commits everything atomically.

Usage: `/dgs:consolidate-ideas 1 3 17` (consolidate ideas 1, 3, and 17)
Usage: `/dgs:consolidate-ideas` (interactive: select from pending ideas)
Usage: `/dgs:consolidate-ideas 1 3 --title "Unified API resilience"` (with explicit title)

**`/dgs:find-related-ideas [id] [--threshold high|medium|low]`**
Find pending ideas related to a selected idea using multi-signal scoring, then optionally flow into consolidation.

- Without ID: lists pending ideas for interactive anchor selection
- Uses three scoring signals:
  - **Tag overlap** (algorithmic): Jaccard similarity of idea tags
  - **Semantic similarity** (AI): Whether ideas describe the same or overlapping problem/solution space
  - **Implementation overlap** (AI): Whether implementing both ideas would touch the same code modules
- Project context files (`codebase/`, `docs/product/`) inform implementation overlap scoring when available
- Composite scoring: ANY HIGH signal = HIGH match; 2+ MEDIUM = HIGH; ANY MEDIUM = MEDIUM; ANY LOW = LOW
- Results displayed grouped by match level (HIGH/MEDIUM/LOW) with per-idea reasoning explaining why each idea is related
- After viewing results, you can select ideas to consolidate with the anchor idea -- no separate command needed
- `--threshold`: Filter results by minimum match level (default: `medium`)
  - `high`: Only high-confidence matches
  - `medium`: High and medium matches (default)
  - `low`: All matches including low-confidence

Usage: `/dgs:find-related-ideas 42` (find ideas related to #42)
Usage: `/dgs:find-related-ideas` (interactive: select anchor idea)
Usage: `/dgs:find-related-ideas 42 --threshold low` (show all matches including low)

**`/dgs:undo-consolidation [id]`**
Undo a previous consolidation by restoring source ideas to pending and deleting the consolidated result idea.

- Without ID: lists pending ideas with `consolidated_from` for interactive selection
- Validates the target idea is a consolidated result (has `consolidated_from` field in frontmatter)
- Shows the consolidated idea and all source ideas that will be restored, then asks for confirmation
- Restores each source idea from `consolidated/` back to `pending/` state
- Removes the `consolidated_into` reference from each restored source idea
- Deletes the consolidated result idea
- Commits atomically: `ideas: undo consolidation #id - title`
- After undo, you can re-consolidate differently with `/dgs:consolidate-ideas`

Usage: `/dgs:undo-consolidation 5` (undo consolidation of idea #5)
Usage: `/dgs:undo-consolidation` (interactive: select from consolidated ideas)

### Documents & Search

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:add-doc <file>` | Add a supporting document (PDF, image, spreadsheet) | Attach reference material to an idea, spec, or product |
| `/dgs:list-docs` | List all supporting documents by scope | See what reference material exists |
| `/dgs:remove-doc` | Remove a document | Clean up outdated references |
| `/dgs:search <query>` | Search across ideas, specs, docs, and projects | Find anything in the planning directory |

#### Documents & Search Details

**`/dgs:add-doc <file>`**
Add a supporting document to a scoped docs/ directory with text extraction.

- `--scope <product|idea|spec>`: Target where the document belongs
- `--scope-id <id>`: Attach to a specific idea or spec by ID

Usage: `/dgs:add-doc architecture.pdf` (auto-detects scope from context)
Usage: `/dgs:add-doc diagram.png --scope idea --scope-id 5` (attach to idea #5)
Usage: `/dgs:add-doc summary.md --scope product` (add as product-level context)

**`/dgs:remove-doc`**
Remove a document or move it to a different scope.

- `--move`: Move a document to a different scope instead of deleting

Usage: `/dgs:remove-doc` (prompts to select and remove)
Usage: `/dgs:remove-doc --move` (reorganize without deleting)

**`/dgs:search <query>`**
Search across ideas, specs, docs, and projects with fuzzy keyword matching.

- `--ideas-only`: Search only ideas
- `--specs-only`: Search only specs
- `--docs-only`: Search only documents
- `--include-rejected`: Include rejected ideas in results
- `--tags TAG1,TAG2`: Filter search results by tags

Usage: `/dgs:search "auth tokens"` (search all content)
Usage: `/dgs:search authentication --ideas-only` (search only ideas)
Usage: `/dgs:search review --include-rejected --tags api` (broad search with filters)

**Product docs as context:** Documents added with `--scope product` are automatically loaded as context by several workflows:
- **`/dgs:write-spec`** and **`/dgs:plan-phase`** load all markdown files from `.planning/docs/product/`
- **`/dgs:new-project`** and **`/dgs:new-milestone`** load `ARCHITECTURE.md` and `PRODUCT-SUMMARY.md` from `.planning/docs/product/` (if they exist) to inform research and roadmap creation

This means uploading a target architecture or product summary via `/dgs:add-doc --scope product` will automatically inform spec drafting, phase planning, project initialization, and milestone creation.

### Project Management

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:list-projects` | Show all projects with status and repos | Overview of the product's projects |
| `/dgs:switch-project [name]` | Switch active project context | Working on a different project |
| `/dgs:complete-project` | Mark the current project as completed | All milestones are done |

### Multi-Repo Management

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:init-product` | Initialize planning repo structure | Setting up multi-repo management |
| `/dgs:add-repo [name]` | Register a sibling repo | Adding a new repo to the project |
| `/dgs:remove-repo [name]` | Unregister a repo from REPOS.md | Removing a repo from the product |
| `/dgs:overlap-check` | Show repos touched by multiple active projects | Detect potential cross-project conflicts |
| `/dgs:health` | Validate repo reachability and config | After setup or when repos seem unreachable |

---

## Configuration Reference

DGS stores product-level settings in `dgs.config.json` (in your planning root). This file is shared across all projects in the product. Configure during `/dgs:init-product` or `/dgs:new-project`, and update later with `/dgs:settings`.

Review API keys are stored separately in `review-keys.json` (see [Cross-LLM Review](#cross-llm-review) below).

### Full dgs.config.json Schema

```json
{
  "mode": "interactive",
  "depth": "standard",
  "model_profile": "balanced",
  "planning": {
    "commit_docs": true,
    "search_gitignored": false
  },
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true,
    "nyquist_validation": true,
    "codereview": false
  },
  "git": {
    "branching_strategy": "none",
    "base_branch": "main",
    "phase_branch_template": "dgs/{project}/phase-{phase}-{slug}",
    "milestone_branch_template": "dgs/{project}/{milestone}-{slug}"
  }
}
```

### Core Settings

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `mode` | `interactive`, `yolo` | `interactive` | `yolo` auto-approves decisions; `interactive` confirms at each step |
| `depth` | `quick`, `standard`, `comprehensive` | `standard` | Planning thoroughness: 3-5, 5-8, or 8-12 phases |
| `model_profile` | `quality`, `balanced`, `budget` | `balanced` | Model tier for each agent (see table below) |

### Planning Settings

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `planning.commit_docs` | `true`, `false` | `true` | Whether `.planning/` files are committed to git |
| `planning.search_gitignored` | `true`, `false` | `false` | Add `--no-ignore` to broad searches to include `.planning/` |

> **Note:** If `.planning/` is in `.gitignore`, `commit_docs` is automatically `false` regardless of the config value.

### Workflow Toggles

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `workflow.research` | `true`, `false` | `true` | Domain investigation before planning |
| `workflow.plan_check` | `true`, `false` | `true` | Plan verification loop (up to 3 iterations) |
| `workflow.verifier` | `true`, `false` | `true` | Post-execution verification against phase goals |
| `workflow.nyquist_validation` | `true`, `false` | `true` | Validation architecture research during plan-phase; 8th plan-check dimension |
| `workflow.codereview` | `true`, `false` | `false` | 3-pass, 9-agent multi-agent code review after each plan execution. Auto-fixes low-risk issues in a separate commit. |

Disable these to speed up phases in familiar domains or when conserving tokens.

### Cross-LLM Review

Review API keys are stored in `review-keys.json` (in your planning root), separate from the main config file. This file is created automatically during `/dgs:init-product` and is gitignored by default.

**review-keys.json:**
```json
{
  "openai": {
    "api_key": "$OPENAI_API_KEY",
    "model": "gpt-5-mini"
  },
  "gemini": {
    "api_key": "$GEMINI_API_KEY",
    "model": "gemini-2.5-flash"
  },
  "max_rounds": 3
}
```

Edit this file directly to configure review keys. Keys can be literal values or environment variable references (prefixed with `$`). Review keys are NOT configured through `/dgs:settings` -- the settings workflow shows their status (set/not set) but does not prompt for changes.

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `openai.api_key` | API key string or `$ENV_VAR` | `""` | OpenAI API key for spec review |
| `openai.model` | Model ID string | `gpt-5-mini` | OpenAI model used for review |
| `gemini.api_key` | API key string or `$ENV_VAR` | `""` | Gemini API key for spec review |
| `gemini.model` | Model ID string | `gemini-2.5-flash` | Gemini model used for review |
| `max_rounds` | Integer | `3` | Maximum review-feedback rounds before convergence |

> **Note:** If no API keys are configured, `/dgs:write-spec` skips the cross-LLM review step entirely. `review-keys.json` is gitignored by default during `/dgs:init-product` to prevent accidental secret commits.

### Git Branching

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `git.branching_strategy` | `none`, `phase`, `milestone` | `none` | When and how branches are created |
| `git.base_branch` | Branch name string | `main` | Integration target for all merge operations |
| `git.phase_branch_template` | Template string | `dgs/{project}/phase-{phase}-{slug}` | Branch name for phase strategy |
| `git.milestone_branch_template` | Template string | `dgs/{project}/{milestone}-{slug}` | Branch name for milestone strategy |

**Branching strategies explained:**

| Strategy | Creates Branch | Scope | Best For |
|----------|---------------|-------|----------|
| `none` | Never | N/A | Solo development, simple projects |
| `phase` | At each `execute-phase` | One phase per branch | Code review per phase, granular rollback |
| `milestone` | At first `execute-phase` | All phases share one branch | Release branches, PR per version |

**Template variables:** `{phase}` = zero-padded number (e.g., "03"), `{slug}` = lowercase hyphenated name, `{milestone}` = version (e.g., "v1.0"), `{project}` = project slug (e.g., "checkout").

**Base branch:** The `base_branch` setting controls which branch DGS uses as the integration target. All branch creation (`git checkout $BASE_BRANCH && git checkout -b $NEW_BRANCH`) and merge operations (`complete-milestone` merges into `$BASE_BRANCH`) use this value. Defaults to `main`. Set to `develop`, `staging`, or any branch that matches your release workflow. Configured during `/dgs:new-project` or via `/dgs:settings`.

**Project-scoped branch names:** The `{project}` template variable ensures branches from different projects never collide. With the default template and project slug "checkout", Phase 3 "auth" creates branch `dgs/checkout/phase-03-auth`. If your config uses older templates without `{project}`, `/dgs:settings` will suggest updating them. Templates without `{project}` continue to work — backwards compatibility is preserved.

### Conflict Resolution

When `/dgs:complete-milestone` merges branches back to the base branch, merge conflicts can occur — especially with the `phase` strategy where multiple branches are merged sequentially.

DGS handles this automatically:

1. **Detection** — Identifies conflicted files and maps them to owning repos
2. **Classification** — Each conflict hunk is classified as one of four types:
   - **ADDITIVE** — One side adds new content (resolved automatically with HIGH confidence)
   - **DELETION** — One side removes content (keeps content unless plan context says otherwise)
   - **STRUCTURAL** — Import/export reorganization (combines with deduplication)
   - **DIVERGENT** — Both sides changed the same code differently (may need your input)
3. **Resolution** — Per-hunk strategy selection based on classification and plan context
4. **Escalation** — LOW-confidence resolutions are presented to you with:
   - The diff and conflict markers
   - Plan tasks that touched the file
   - A proposed resolution
   - Options: **accept**, **reject with hint** (e.g., "keep the version from phase 3"), or **abort**
5. **Verification** — After resolution, available tests/linting run on affected files. Failed verification triggers rollback.
6. **Audit trail** — Every resolution is recorded in `RESOLUTIONS.md` and the milestone's `RESOLUTION-REPORT.md`

**Cascading conflicts:** When multiple phase branches are merged sequentially, learnings from earlier merges (which files conflicted, what strategies worked) are passed to later merges to improve confidence.

**Semantic conflict warnings:** Even when a merge succeeds textually, DGS flags cases where both branches modified behavior in the same domain — these may need integration testing.

**CLI tools** (for scripting or debugging):

| Command | What it Does |
|---------|--------------|
| `dgs-tools merge-conflicts detect` | List conflicted files with owning repos |
| `dgs-tools merge-conflicts context <file>` | Assemble resolution context for a file |
| `dgs-tools merge-conflicts resolved <file>` | Record a resolution outcome |
| `dgs-tools merge-conflicts summary` | Aggregate resolution report |
| `dgs-tools conflict-agent run` | Run full automated resolution |
| `dgs-tools conflict-agent resolve-file <file> [--hint "text"]` | Resolve a single file with optional hint |

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

The `--no-check` flag omits the `audit-milestone` and `complete-milestone` steps from the generated job, useful when you want to run phases without the final audit cycle.

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

---

## Usage Examples

### New Project (Full Cycle)

```bash
claude --dangerously-skip-permissions
/dgs:new-project            # Answer questions, configure, approve roadmap
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
/dgs:new-project --auto @prd.md   # Auto-runs research/requirements/roadmap from your doc
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
/clear
/dgs:discuss-phase 1        # Normal phase workflow from here
```

### Existing Codebase

```bash
/dgs:map-codebase           # Map all registered repos (per-repo docs + unified synthesis)
/dgs:new-project            # Questions focus on what you're ADDING
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

Set `commit_docs: false` during `/dgs:new-project` or via `/dgs:settings`. Add `.planning/` to your `.gitignore`. Planning artifacts stay local and never touch git.

### DGS Update Overwrote My Local Changes

Since v1.17, the installer backs up locally modified files to `dgs-local-patches/`. Run `/dgs:reapply-patches` to merge your changes back.

### Subagent Appears to Fail but Work Was Done

A known workaround exists for a Claude Code classification bug. DGS's orchestrators (execute-phase, quick) spot-check actual output before reporting failure. If you see a failure message but commits were made, check `git log` -- the work may have succeeded.

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

---

## Product File Structure

For reference, here is what DGS creates. Product-level files are shared across all projects. Each project gets its own subdirectory under `.planning/`.

```
.planning/                          # PRODUCT LEVEL
  dgs.config.json                   # Product-wide configuration (shared across projects)
  review-keys.json                  # Review API keys (gitignored, edit directly)
  PROJECTS.md                       # Auto-generated project registry
  REPOS.md                          # Multi-repo registry (../repo-name paths)
  MILESTONES.md                     # Completed milestone archive
  ideas/                            # Idea pipeline (product-wide)
    pending/                        #   Ideas awaiting action
    done/                           #   Ideas consumed by specs
    rejected/                       #   Ideas explicitly declined
  specs/                            # Spec pipeline (product-wide)
    draft/                          #   Specs in progress
    review/                         #   Specs in cross-LLM review
    final/                          #   Finalized specs ready for projects
  docs/                             # Supporting documents (product-wide)
    product/                        #   Product-level docs (loaded as context by workflows)
      ARCHITECTURE.md               #     Target architecture (used by new-project, new-milestone)
      PRODUCT-SUMMARY.md            #     Product summary (used by new-project, new-milestone)
      *.md                          #     All markdown files loaded by write-spec, plan-phase
    ideas/                          #   Idea-scoped docs (move to done/ with idea)
    specs/                          #   Spec-scoped docs
  codebase/                         # Codebase mapping (from /dgs:map-codebase)
    <repo-name>/                    #   Per-repo subdirectory (one per registered repo)
      STACK.md                      #     Technology stack
      ARCHITECTURE.md               #     System architecture
      STRUCTURE.md                  #     Project structure
      CONVENTIONS.md                #     Code conventions
      TESTING.md                    #     Testing approach
      INTEGRATIONS.md               #     External integrations
      CONCERNS.md                   #     Known issues and risks
    ARCHITECTURE.md                 #   Unified (synthesized from all repos)
    STACK.md                        #   Unified
    STRUCTURE.md                    #   Unified
    CROSS-REPO.md                   #   Cross-repo analysis (2+ repos)
  milestones/                       # Milestone archive (versioned)
    v1.0-ROADMAP.md                 #   Archived roadmap
    v1.0-REQUIREMENTS.md            #   Archived requirements
    v1.0-MILESTONE-AUDIT.md         #   Audit results
    v1.0-phases/                    #   Archived phase directories
  jobs/                             # Milestone job pipeline
    pending/                        #   Jobs awaiting execution
    in-progress/                    #   Currently executing jobs
    completed/                      #   Finished jobs with summaries

  <project-slug>/                   # PROJECT LEVEL (one per project)
    PROJECT.md                      #   Project vision and context (always loaded)
    REQUIREMENTS.md                 #   Scoped requirements with IDs
    ROADMAP.md                      #   Phase breakdown with status tracking
    STATE.md                        #   Decisions, blockers, session memory
    research/                       #   Domain research from /dgs:new-project
    todos/                          #   Project-specific todos
      pending/                      #     Captured tasks awaiting work
      done/                         #     Completed todos
    debug/                          #   Active debug sessions
    quick/                          #   Quick task artifacts
      N-task-slug/                  #     Per-task plan and summary
    phases/                         #   Phase directories
      XX-phase-name/
        XX-YY-PLAN.md              #     Atomic execution plans
        XX-YY-SUMMARY.md           #     Execution outcomes and decisions
        CONTEXT.md                 #     Your implementation preferences
        RESEARCH.md                #     Ecosystem research findings
        VERIFICATION.md            #     Post-execution verification results
        XX-VALIDATION.md           #     Test coverage contract (Nyquist layer)
```
