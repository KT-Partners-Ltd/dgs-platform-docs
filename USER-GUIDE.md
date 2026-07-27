# DGS User Guide

A detailed reference for workflows, troubleshooting, and configuration. For a linear first-time walkthrough, see [GETTING-STARTED.md](GETTING-STARTED.md); for quick-start setup, see the [README](../README.md).

---

## Sub-Guides

Detailed reference for specific topics lives in dedicated sub-documents:

- **[Command Reference](COMMAND-REFERENCE.md)** -- Every DGS command with flags, usage examples, and detailed behavior
- **[Configuration Guide](CONFIGURATION-GUIDE.md)** -- config.json schema, workflow toggles, cross-LLM review, conflict resolution
- **[Milestone Jobs & Execution Lifecycle](MILESTONE-JOBS-GUIDE.md)** -- Quick workflows, milestone jobs, phase audit, model profiles
- **[Setup Guide](SETUP-GUIDE.md)** -- Multi-repo setup, codebase mapping, product file structure
- **[Multi-User Approach Guide](MULTI_USER_APPROACH_GUIDE.md)** -- Coordinating DGS across multiple developers, four-eyes completion governance
- **[Context Monitor](context-monitor.md)** -- Context window usage monitoring and optimization

---

## Table of Contents

- [Workflow Diagrams](#workflow-diagrams)
- [Context Tiers](#context-tiers)
- [Usage Examples](#usage-examples)
- [Querying the Artifact Graph](#querying-the-artifact-graph)
- [Threads](#threads)
- [Troubleshooting](#troubleshooting)
- [Recovery Quick Reference](#recovery-quick-reference)

---

## Workflow Diagrams

### Full Product Lifecycle

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │                             PRODUCT SETUP                              │
  │  /dgs:init-product                                                     │
  │  Creates config.json, REPOS.md, PROJECTS.md at the planning-repo root  │
  │  /dgs:add-repo to register source repos                                │
  └────────────────────────────────────┬───────────────────────────────────┘
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
     ideas/                       │                           │
       │                          │                           │
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
                           --auto <spec-id>
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
         │            │
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

### Dependency Scanning Workflow

Run on demand at any point in the project lifecycle — no roadmap entry or phase required.

```
[Dev]
  │
  ├─► /dgs:package-scan
  │      │
  │      ├─ Enumerate targets: every repo in REPOS.md + product root
  │      ├─ Detect ecosystems per target (manifest-file inspection)
  │      ├─ Expand monorepos: npm/pnpm/Yarn workspaces, Maven modules, go.work
  │      ├─ Select tool: Snyk → OSV-Scanner → native (ecosystem-specific)
  │      ├─ Run tool per target, collect JSON output
  │      ├─ Normalise findings to canonical shape (test-gate contract)
  │      ├─ Resolve plan provenance via `git log -S <package> -- <manifest>`
  │      └─ Commit report to active phase / active milestone / project root
  │
  └─► Developer reviews report
         │
         ├─ Triage findings by severity (critical → high → medium → low)
         ├─ Apply suggested fix commands (`npm install ...`, `pip install -U ...`, etc.)
         └─ (Future) /dgs:plan-test-gaps consumes the findings YAML to generate fix phases
```

**When to run:**
- Before a release — surface any new critical vulnerabilities before shipping.
- After dependency changes — catch regressions introduced by `npm install`, `go mod tidy`, etc.
- Ad-hoc — no state change to your project; the report file is the only artefact.

**Zero-config mode:** with nothing installed beyond `npm`, `/dgs:package-scan` falls back to `npm audit` for Node, `pip-audit` for Python (if installed), `govulncheck` for Go (if installed), and `bundler-audit` for Ruby (if installed). Install OSV-Scanner (a single binary) or configure a Snyk token for full multi-ecosystem coverage in one invocation.

See the [Command Reference](COMMAND-REFERENCE.md#testing--dependency-scanning) and [Configuration Reference](CONFIGURATION-GUIDE.md#testing--package-scanning) for flag details and config keys.

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
| `/dgs:health [--repair]` | Validate planning-repo-root integrity |
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
| `/dgs:new-milestone [name] [--adhoc]` | Start next version cycle; `--adhoc` creates a lightweight container milestone |
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
| `/dgs:abandon-milestone` | Discard an ad-hoc container milestone, restore planning docs |
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
/dgs:new-project                 # Create PROJECT.md from your doc
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
/dgs:new-project            # Create project from spec
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
/dgs:code-review branch     # Milestone-level code audit — catches cross-plan issues the per-plan gate can't see
/dgs:audit-milestone        # Check requirements coverage, detect stubs
/dgs:plan-milestone-gaps    # If audit found gaps, create phases to close them
/dgs:complete-milestone     # Archive, tag, done
```

### On-Demand Code Review

`/dgs:code-review` is the human-invoked complement to the automatic per-plan codereview gate — run it yourself at lifecycle boundaries the per-plan gate never sees:

```bash
/dgs:code-review                  # context-aware default: branch when focused/clean, staged otherwise
/dgs:code-review branch           # before /dgs:complete-quick — whole-quick review
/dgs:code-review branch           # before /dgs:complete-milestone — run in the milestone worktree
/dgs:code-review staged           # after manual/skip-DGS edits
/dgs:code-review https://github.com/owner/repo/pull/123   # review any PR on a registered repo
```

**Local mode** (staged/unstaged/branch/file) aggressively auto-fixes CRITICAL/HIGH/MEDIUM findings — on `branch` scope the fixes land as one `fix(review):` commit, otherwise they stay in your working tree. **PR mode** (a PR URL) is comment-only — it never edits files and requires an authenticated `gh`.

See [Command Reference](COMMAND-REFERENCE.md#brownfield--utilities-details) for the full scope, mode, and output details.

### Security Review

`/dgs:security-review` is the security lens, run after `/dgs:code-review` has settled the diff. Where code-review reads the diff *inward* for line-level correctness, security-review reads it *outward* as **new attack surface**: what does this change let an attacker reach that they could not reach before — a moved trust boundary, a newly-reachable dangerous sink, a secret now flowing somewhere new?

It reports and routes; it never edits source and never auto-fixes. Findings are non-blocking by convention — whether one should stop a completion is your call.

Each finding is scored on two independent axes. **Severity** is how bad it is *if the vulnerable sink is actually reached* (`info` → `critical`); **`reachability_confidence`** (`confirmed`, `probable`, or `unproven`) is whether the lens could demonstrate that untrusted path from the diff itself. They are kept separate on purpose: an unproven-but-catastrophic sink keeps its high severity and is flagged `reachability_confidence: unproven` rather than being quietly downgraded — so you triage on impact first and confirm reachability as a second step. The lens pins `model: opus`, so its verdicts don't drift with whichever model you happen to invoke the command from.

Recommended cadence is a **convention, not a trigger**: run it at end-of-milestone over the accumulated diff, or opt in on a phase or quick that touches security-sensitive surface (auth, network handlers, deserialization, secrets). Deliberately *not* every phase — a gate that always fires with nothing to say trains you to ignore it.

```bash
/dgs:security-review                  # default: branch scope, the settled change
/dgs:security-review staged           # what's staged right now
/dgs:security-review https://github.com/owner/repo/pull/123   # a PR, fork-safe and read-only
```

### Claim-Refutation Review

`/dgs:adversarial-review` is the final trust gate, run after `/dgs:code-review`: canonical order is audit-phase → code-review → security-review → adversarial-review. Where code-review asks "is this code correct?", adversarial-review asks "is the CLAIM that this works actually true?" — it attacks the claims of the finished post-fix artifact by executing code (running tests, curling endpoints, invoking the CLI) rather than reading source. Refuted claims route to `/dgs:quick` or a gap phase rather than being fixed inline.

```bash
/dgs:adversarial-review    # final trust gate — refute the green before completing
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

## Querying the Artifact Graph

DGS maintains a SQLite **artifact index** (`node:sqlite`) built from the frontmatter of every planning artifact plus a `links` graph. It is a rebuildable cache, never the source of truth: it auto-builds and reconciles itself, and every consumer falls back byte-identically to the filesystem when the index is absent, stale, or unsupported (Node < 22.13). No command ever hard-fails because of it — it transparently speeds up existing consumers like `/dgs:progress`, `/dgs:list-ideas`, and `/dgs:list-jobs` with identical output.

Two commands expose the index directly:

### Ask a question — `/dgs:query`

```
/dgs:query "which ideas are still pending?"
/dgs:query "what phases does milestone v28.0 have?" --sql
/dgs:query "specs that reference idea 12" --limit 20
```

Your natural-language question is translated — entirely in your local session, with no external model or telemetry — into a single read-only `SELECT`, executed through an engine-level safety guard (SELECT + safe-PRAGMA only, injected `LIMIT`, a 5s statement interrupt, parameter binding), and rendered in the familiar DGS list/progress style with a truncation footer. A non-SELECT or DDL query is rejected fail-closed with the offending SQL shown. Pass `--sql` to see the generated query, `--limit N` to cap rows, and ask follow-up questions to refine conversationally. It also answers reverse-link and full-chain traceability questions (what references idea N; idea → spec → milestone → phases).

When `query.enabled` is set to `false`, `/dgs:query` still generates and shows the SQL but does not auto-execute it — useful for headless or CLI-only use. See [Configuration Reference](CONFIGURATION-GUIDE.md#query) for the toggle.

### Check graph integrity — `/dgs:integrity`

```
/dgs:integrity
/dgs:integrity --quiet
```

Reports integrity problems grouped by class — dangling `id-ref`/link targets, orphaned artifacts, and broken `milestone` references — with per-class and total counts. A clean graph prints `✓ no integrity issues`. It is strictly read-only and virtual-aware: virtual targets (`quick`, `fast`) are never false-flagged, while a `milestone` that genuinely resolves to nothing is reported. It exits nonzero when any finding exists, so it is ready to wire into CI or a pre-push hook (it is deliberately not part of DGS's own commit gate in v28.0). Use `--quiet` for summary counts only. When the index is absent it degrades to a filesystem fallback covering dangling and broken-milestone checks.

---

## Threads

A **thread** is an organizing topic ABOVE ideas — a committed, team-visible, indexed home for a line of work that generates ideas and todos and accumulates decisions across sessions and teammates. Where an idea is a single build-bound capture, a thread never itself graduates to a spec or phase; its output IS the ideas/todos it spawns, plus a decisions ledger that nothing else in DGS records.

```
/dgs:add-thread "Payments v2 rework"
/dgs:list-threads
/dgs:resume-thread payments-v2-rework
/dgs:promote-idea 42                    # an existing idea grows into an anchoring thread
/dgs:backfill-threads                   # retrofit threads onto an existing idea backlog
```

### Thread vs idea

An idea is a single, disposable capture that develops linearly toward a spec (idea → discuss → research → spec). A thread OWNS a family of child ideas and todos and persists as long as the line of work does — it is the topic, not a single unit of work. Use `/dgs:promote-idea` when an idea turns out to be the anchor of an ongoing effort rather than a one-shot capture.

### Document anatomy

Each thread file (`threads/<id>.md`) has five sections: **Goal**, **Context**, **Decisions Ledger**, **Next Steps**, **Log**.

- **Goal** and **Next Steps** are steward prose — seeded empty at create, and rewritten only by `thread compact`.
- **Context** is normally an append-only entry stream (each `append-context` call adds an attributed, UTC-timestamped entry, oldest first) — but, like Goal and Next Steps, it CAN be consolidated into clean prose by the guard-railed `thread compact --context <text>`, which overwrites the whole Context section with the given text.
- **Decisions Ledger** and the activity **Log** are STRICTLY append-only. `compact` never touches them — no matter which of `--goal`/`--context`/`--next-steps` you pass, the Decisions Ledger and Log are left byte-identical. They are the one part of a thread that is pure history: nothing consolidates or rewrites them, ever.

So the distinction is: Goal, Context, and Next Steps are all compact-consolidatable (Context is simply the one that is *also* a normal append target day-to-day); the Decisions Ledger and Log are append-only, full stop. No derived child state (idea/todo counts, statuses) is ever written into the thread file itself — that's always computed live from the index (see Lineage & roll-ups below).

`thread compact` is a coordinate-first, solo-moment operation: it fetches first and refuses to run against un-pulled remote changes to the same thread file (override with `--force`), because a prose rewrite can't be safely unioned the way append-only entries can.

### Identity & lifecycle

A thread's `id` is a slug — immutable once created, auto-derived from the title when you don't pass `--id` explicitly (never an integer, unlike ideas/todos). Lifecycle is **open** → **dormant** → **closed** (and back to open on reopen). Appends to a closed thread are rejected outright — the file is left untouched — with a reopen hint pointing at `dgs-tools thread set-status <id> open`.

### Lineage & roll-ups

Every child idea or todo owns AT MOST ONE forward `thread` edge (cardinality one) — a thread itself owns no forward edge back to its children. Roll-up counts and status are always live reverse-index queries, never text stored in the thread file. `/dgs:check-todos --by-thread` groups todos under their owning thread (the engine-level `ideas list --by-thread` does the equivalent grouping for ideas); `/dgs:promote-idea` stamps an idea's edge onto a brand-new anchoring thread in one atomic commit, retaining the idea's own status and lifecycle unchanged.

### Dashboard visibility

Threads are browsable as a first-class type in the local DGS dashboard — the threads view lists each thread with its lifecycle status and its live child roll-up (idea/todo counts), the same reverse-junction counts `/dgs:list-threads` surfaces.

### Similarity guard

`/dgs:add-thread` runs a create-time similarity pass before minting a thread: a HIGH-confidence match against an existing thread blocks the create and offers append/link instead (override with `--force-new` to mint a distinct thread anyway); a HIGH match against a CLOSED thread routes you to reopen it instead of silently forking a duplicate. A MEDIUM match is informational only — create proceeds with a non-blocking heads-up.

### Resume across sessions

`/dgs:resume-thread` reloads a thread's goal, decisions ledger, next steps, and open todos into the session — the lightweight thread-scale equivalent of `/dgs:resume-work`.

### Multi-user convergence

Threads are DGS's committed multi-user primitive — concurrent appends from multiple clones converge over git with no manual conflict surgery. See [Multi-User Approach Guide](MULTI_USER_APPROACH_GUIDE.md) for the mechanics.

---

## Troubleshooting

### "Project already initialized"

You ran `/dgs:new-project` but `PROJECT.md` already exists. This is a safety check. If you want to start over, clear the planning folder.

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

Set `commit_docs: false` via `/dgs:settings` (defaults to `true`).

### DGS Update Overwrote My Local Changes

Since v1.17, the installer backs up locally modified files to `dgs-local-patches/`. Run `/dgs:reapply-patches` to merge your changes back.

### Subagent Appears to Fail but Work Was Done

A known workaround exists for a Claude Code classification bug. DGS's orchestrators (execute-phase, quick) spot-check actual output before reporting failure. If you see a failure message but commits were made, check `git log` -- the work may have succeeded.

### Ad-hoc Milestones

An ad-hoc container milestone is a lightweight, worktree-isolated milestone with no spec or roadmap — just a branch that batches small work behind an atomic, no-leak window.

Use one when you want a run of quicks and fasts to ship together under a single version (or be discarded as a unit) without leaking changes onto main.

Lifecycle:

1. `/dgs:new-milestone --adhoc "Experiments" --version v0.1` snapshots the base ref and opens the milestone (creates the milestone worktree and branch).
2. Quicks and fasts run during the milestone auto-join the milestone branch unchanged — no extra flags.
3. `/dgs:complete-milestone` ships the batch as one vX.Y (a relaxed, quicks-only completion gate applies), or `/dgs:abandon-milestone` restores planning docs from the snapshot behind a `--confirmed` gate.

The `adhoc` marker is visible in `/dgs:progress` and the completion preamble.

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

### PR-Based Completion (`git.completion_mode: pr`)

By default, completing a quick or milestone merges to `base_branch` locally. On branch-protected or review-gated repos, switch completion to pull requests:

```
dgs-tools config set git.completion_mode pr
```

The key lives in the tracked `config.json` (valid values: `merge`, `pr`; unset means `merge`). In `pr` mode the GitHub CLI (`gh`) must be installed and authenticated (`gh auth login`) — **only** this mode needs it; the default `merge` path never invokes `gh`.

The lifecycle, end to end:

1. **Open** — `/dgs:complete-quick` rebases onto base, pushes the branch with `--force-with-lease`, and opens a PR (one per touched repo): `PR opened: <url>`. The quick parks at `pr_open`; its worktree stays.
2. **Update** — push review fixes by committing in the worktree and re-running `/dgs:complete-quick`: the same PR gets the new head (`PR updated — head pushed`). A title or body you edited on GitHub is never overwritten.
3. **Reap** — after the PR merges on GitHub, re-run `/dgs:complete-quick` (or sweep all merged quicks at once with `dgs-tools reap-quicks`): DGS verifies the merge via `gh`, pulls base, and removes the worktree and branch.

Edge cases:

- **`gh` outage during reap:** DGS fails closed — `Couldn't reach GitHub. If you know it merged, re-run with --merged`. The `--merged` flag asserts the merge and reaps without `gh`.
- **PR closed without merging:** the reap refuses (your work is unmerged); re-run with `--confirm-cleanup` to remove the worktree anyway.
- **Multi-repo:** one PR per touched repo; nothing reaps until *every* repo's PR is merged.

The four-eyes reviewer reviews the DGS-opened PR with `/dgs:code-review <PR URL>` — it resolves stale review threads and posts one consolidated comment, never editing the author's branch.

Milestones follow the same model through `/dgs:complete-milestone`, with archival happening at reap — never at PR open. Full model: [Completion Modes: Merge vs PR](GIT-WORKFLOW.md#completion-modes-merge-vs-pr); sweep command: [reap-quicks](COMMAND-REFERENCE.md) in the Command Reference.

### Per-Console Context

The active context — the milestone or one of the standalone quicks that DGS commands operate on — is a property of your **console**, not a global setting. Multiple consoles can each drive a different milestone or quick **in parallel** without clobbering each other: resolution, commit routing, and the status bar all track the focus of the console they run in.

Resolution precedence, highest first:

1. `--context <slug>` — a one-shot override on a single command
2. **session binding** — the per-Claude-Code-session focus (`CLAUDE_CODE_SESSION_ID` → `config.local.json` `execution.console_bindings`)
3. `DGS_CONTEXT` — the per-shell environment variable (raw-shell path)
4. the config default (`execution.active_context`)
5. none (product / main)

#### Claude Code sessions (the default — no shell setup)

Inside a Claude Code session, each session binds **its own** focus automatically — there is no `.zshrc` edit and no relaunch. DGS keys the binding on the session's `CLAUDE_CODE_SESSION_ID` (present in every command subprocess) and stores it in `config.local.json` under `execution.console_bindings` (`{ "<session_id>": "<context-slug>" }`), so two Claude Code sessions on the same project never clobber one shared focus pointer:

- **`switch-context <slug>`** binds the current session to that milestone or quick (in-session, immediately).
- **Quick-create** (`/dgs:quick`) binds the new quick to the session that created it.
- **`complete-quick` / `abandon-quick` / `complete-milestone`** unbind the finishing session (and drop every session's binding that pointed at the removed slug).

The status bar shows **each console's own focus** (it resolves through the same session-keyed resolver), so two side-by-side sessions display different contexts. `dgs-tools context` prints `<slug> (session)` when the console is bound this way.

#### Raw-shell users (DGS_CONTEXT)

In a plain terminal (not a Claude Code session), there is no `CLAUDE_CODE_SESSION_ID`, so the per-shell `DGS_CONTEXT` environment variable is the binding path. This works exactly like `KUBECONFIG`, `AWS_PROFILE`, or `pyenv shell`: the binding lives in the shell's environment, and DGS reads it.

**Bind a console with a `dgs-bind` shell function.** A child process cannot mutate its parent shell's environment, so DGS emits the `export` line for the shell to `eval`. Add this helper to your shell rc (`~/.zshrc` / `~/.bashrc`):

```sh
dgs-bind() { export DGS_CONTEXT="$1"; }
```

Then bind this console to a context. `switch-context --print` is **read-only** — it emits `export DGS_CONTEXT=<slug>` and writes nothing to config:

```sh
eval "$(dgs-tools switch-context my-quick-slug --print)"
# this console now drives `my-quick-slug`; other consoles are unaffected
```

(Or just `dgs-bind my-quick-slug` if you already know the slug.)

**Release on finalize.** When you `complete-quick` or `abandon-quick` a quick this console is bound to, DGS prints an `unset DGS_CONTEXT` advisory so the console stops pointing at a removed worktree. Clear it with:

```sh
unset DGS_CONTEXT
```

**One-shot override.** To run a single command against a different context without rebinding the console, pass `--context <slug>`. The flag wins over `DGS_CONTEXT` for that command only, and does **not** overwrite the env var — so a stale `--context` (a slug with no live worktree) warns and falls through to your inherited `DGS_CONTEXT`:

```sh
dgs-tools quick "fix typo" --context other-milestone
```

**Check your binding.** `dgs-tools context` prints the resolved context and where it came from as `<slug> (<source>)` — e.g. `quick-A (session)` (bound to this Claude Code session via `console_bindings`), `quick-A (env)` (bound via `DGS_CONTEXT`), `quick-A (flag)` (a one-shot `--context`), or `v25.1 (default)` (no per-console binding — using the shared config default / milestone). Use it whenever you are unsure which milestone or quick this console is about to act on:

```sh
dgs-tools context
# quick-A (env)
```

**Unbound warning.** A console with no `DGS_CONTEXT` (and no `--context`) prints `Notice: unbound — defaulting to '<milestone>'` once before a context-sensitive command (`complete-quick`, `abandon-quick`, `reap-quicks`, `quick`, `fast`, `milestone …`), so you never silently act on the milestone when you meant a quick. Bind with `dgs-bind <slug>` or pass `--context <slug>` to silence it.

**Completion is context-typed.** `complete-quick` refuses a milestone-bound console and `complete-milestone` refuses a quick-bound console — each tells you the slug it is bound to and the correct command to run instead. This stops a milestone-bound console from accidentally finalizing through the quick path (and vice versa).

**The cap counts quicks only.** The standalone-quick cap (3) counts quick worktrees only — the milestone context never consumes a slot, so a focused milestone plus three parallel quicks is the expected steady state.

### Stale Milestone Dashboard (shipped milestone shows in-progress)

**Symptom:** After a milestone is archived and tagged, `/dgs:list-projects` still shows a stale in-progress `Phase:` (e.g. `3 of 8`) or a false `executing` status for that project — because its `STATE.md` was never flipped to shipped.

**Going forward this self-corrects:** milestone completion now resets the `Phase:` line automatically. The repair below is only needed for projects already shipped before that fix.

**Repair with `dgs-tools state reconcile-milestone`:**

- **Conservative & idempotent.** It only heals a project *proven* shipped — it requires **both** a `## <version>` heading in `MILESTONES.md` **and** a matching git tag `<version>`. Otherwise it does nothing (`not_shipped`). Re-running an already-correct project is a safe no-op (`already_correct`), so it's safe to run anywhere.
- **What it changes:** `STATE.md` only — frontmatter `status → milestone_shipped` and `progress → 100%`, plus the body's Progress bar, Current focus, Status, Last activity, and the `Phase:` line (reset to a "between milestones" value).
- **Scope limits:** current project only; never touches ROADMAP / MILESTONES / archives / git; does **not** recompute legacy-format progress counters; there is no `--all` mode yet.

**Playbook** (for dashboards that may be silently stale):

1. Update DGS so the command exists: `/dgs:update`
2. For each affected project (safe to run everywhere — it self-detects):
   ```
   dgs-tools switch-project <slug>
   dgs-tools state reconcile-milestone --raw
   ```
3. Verify with `/dgs:list-projects` — shipped projects should read "between milestones", and none should show `executing`.

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
| Shipped milestone shows in-progress/executing | `dgs-tools state reconcile-milestone` (per project) |
