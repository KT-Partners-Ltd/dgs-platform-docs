# Command Reference

> See also: [USER-GUIDE.md](USER-GUIDE.md) for workflow overview and usage examples.

### Command Hierarchy

DGS commands span a range of scope and automation. The hierarchy below orders them from lightest to heaviest — choose the smallest command that fits your task.

| Tier | Command | Scope | Subagents | Best For |
|------|---------|-------|-----------|----------|
| 1 | `/dgs:fast <desc>` | 1-3 files, ~30 lines | None | Typo fixes, config tweaks, one-line changes |
| 2 | `/dgs:quick` | Small feature or bug fix | Planner + executor | Bug fixes, small features, ad-hoc tasks |
| 3 | `/dgs:quick --full` | Medium task needing quality guarantees | Planner + checker + executor + verifier | Tasks where you want plan-checking and post-execution verification without full milestone ceremony |
| 4 | `/dgs:execute-phase <N>` | Full planned phase | Research + planner + checker + executor + verifier | Milestone work with research, planning, and verification |
| 5 | `/dgs:debug [desc]` | Open-ended investigation | Debug agents | Systematic diagnosis when something is broken |

**How to choose:** Start with `/dgs:fast` for trivial edits. If the scope warning fires (more than 3 files or 30 lines), step up to `/dgs:quick`. Use `/dgs:quick --full` when you want verification but don't need milestone ceremony. Use `/dgs:quick --debug` when something is broken and you want investigation before fixing. Reserve `/dgs:execute-phase` for planned work with requirements and roadmap tracking.

**What `--full` adds:** The `--full` flag on `/dgs:quick` enables the plan-checker agent (validates plans achieve the task goal, up to 2 iterations) and the post-execution verifier agent (confirms deliverables match intent). These are the same quality gates used in `/dgs:execute-phase`, applied to an ad-hoc task.

**What `--debug` adds:** The `--debug` flag tells the planner to investigate root cause before fixing. The agent documents findings, may result in no code change if the issue is environmental. Same git mechanics as a regular quick task.

### Core Workflow

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:new-project` | Project identity: questioning → PROJECT.md | Start of a new project (run new-milestone after) |
| `/dgs:new-milestone [name]` | Start next version cycle | After new-project or completing a milestone |
| `/dgs:discuss-phase [N]` | Capture implementation decisions | Before planning, to shape how it gets built |
| `/dgs:plan-phase [N]` | Research + plan + verify | Before executing a phase |
| `/dgs:execute-phase <N>` | Execute all plans in parallel waves | After planning is complete |
| `/dgs:verify-work [N] [--auto]` | Interactive UAT (`--auto` for rubber-stamp pass) | After execution completes |
| `/dgs:audit-phase <phase> [--rerun-failed]` | Automated phase verification (tests + structural inspection) | After execution completes |
| `/dgs:audit-milestone` | Verify milestone met its definition of done | Before completing milestone |
| `/dgs:complete-milestone` | Archive milestone, tag release | All phases verified |

#### Core Workflow Details

**`/dgs:new-project`**
Establish project identity through interactive questioning. Creates PROJECT.md with goals, constraints, and scope. Run `/dgs:new-milestone` after to begin research and planning.

- `--auto @file.md`: Automated init from a PRD or idea document
- `--auto <spec-id>`: Automated init from a finalized spec

Usage: `/dgs:new-project` (interactive questioning flow)
Usage: `/dgs:new-project --auto @prd.md` (from idea document)
Usage: `/dgs:new-project --auto spec-review-config` (from finalized spec)

**`/dgs:new-milestone [name]`**
Start a new milestone cycle for an existing project.

- `--auto <spec-id>`: Derive milestone from a finalized spec without interactive questioning

Usage: `/dgs:new-milestone` (interactive milestone creation)
Usage: `/dgs:new-milestone --auto spec-review-config` (from finalized spec)

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
Merge all milestone work to main and archive the milestone. Manual only — jobs never auto-run this.

For each code repo (sequentially):

1. Pulls latest `base_branch` from remote into main checkout
2. Rebases milestone branch onto `base_branch` in worktree
3. Conflict-agent attempts automatic resolution. If it fails: aborts rebase, provides manual resolution commands
4. Fast-forward merges milestone branch to `base_branch`
5. Pushes to remote
6. Removes milestone branch and worktree

After all code repos merge: updates STATE.md in planning repo to mark milestone complete. Archives phase directories and creates a git tag.

**Multi-repo:** Repos process sequentially (repo 1 then repo 2, etc.). Stops on first failure. Already-merged repos are skipped on re-run (idempotent).

**If rebase conflicts require manual resolution:**

```bash
cd ~/dev/myapp--gsd-v19          # cd to worktree
git rebase main                  # start rebase
# resolve conflicts, then:
git add .
git rebase --continue
# repeat if multiple commits have conflicts
# when done, re-run /dgs:complete-milestone
```

**Job-mode behavior:** When invoked from a milestone job with `<job-mode>silent</job-mode>`, all interactive gates are auto-resolved: scope confirmation is auto-approved, incomplete requirements are acknowledged, phase directories are archived automatically, branch handling is kept for manual review, and tag push is skipped for safety.

Usage: `/dgs:complete-milestone 1.0.0`

See [How Git is Used](GIT-WORKFLOW.md) for a conceptual overview of the rebase-before-merge strategy.

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
| `/dgs:sync [pull\|push\|status]` | Pull from and push to all registered repos | Keep planning and code repos in sync with remote |
| `/dgs:fast <desc>` | Trivial edit with single atomic commit, no subagents | Typo fix, config tweak, one-line change |
| `/dgs:quick` | Ad-hoc task with DGS guarantees | Bug fixes, small features, config changes |
| `/dgs:debug [desc]` | Systematic debugging with persistent state | When something breaks |
| `/dgs:add-todo [desc]` | Capture an idea for later | Think of something during a session |
| `/dgs:check-todos [area]` | List pending todos, optionally filtered by area | Review captured ideas |
| `/dgs:cleanup` | Archive completed quick task directories | Reduce clutter in quick/ directory |
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
- `--debug`: Investigation-focused — agent diagnoses root cause before fixing, documents findings, may result in no code change
- `--main`: Force product-level quick (own worktree off main) even when a milestone is active
- `--fast`: Equivalent to `/dgs:fast` — no subagents, single atomic commit

Usage: `/dgs:quick` (prompts for task description)
Usage: `/dgs:quick fix the login button` (with inline description)
Usage: `/dgs:quick --full add input validation` (with plan-checking and verification)
Usage: `/dgs:quick --debug tests failing after merge` (investigate before fixing)

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
- **`/dgs:new-milestone`** loads `ARCHITECTURE.md` and `PRODUCT-SUMMARY.md` from `.planning/docs/product/` (if they exist) to inform research and roadmap creation

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
