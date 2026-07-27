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
| `/dgs:abandon-milestone` | Discard an ad-hoc milestone, restore planning docs | Abandoning an ad-hoc container |
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
- `--adhoc`: Create a lightweight ad-hoc container milestone (provisional version, snapshot base ref, milestone worktree); quicks & fasts route into it; completion relaxes the readiness gate

Usage: `/dgs:new-milestone` (interactive milestone creation)
Usage: `/dgs:new-milestone --auto spec-review-config` (from finalized spec)
Usage: `/dgs:new-milestone --adhoc "Experiments" --version v0.1` (ad-hoc container milestone)

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

**PR mode (`git.completion_mode: pr`):** instead of steps 4–6, each repo's milestone branch is pushed with `--force-with-lease` and an idempotent PR is opened via `gh`; the milestone parks at `pr_open` with a per-repo PR record (`entry.prs`). Re-run after all PRs merge to reap the worktree and archive — archival happens at reap, never at PR open. Four-eyes governance gates at PR open only. `--merged` asserts the merge without `gh`. `gh` is required only in this mode. See [Completion Modes: Merge vs PR](GIT-WORKFLOW.md#completion-modes-merge-vs-pr).

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

**`/dgs:abandon-milestone`**

Cleanly discard an ad-hoc container milestone. Removes all milestone worktrees and local
branches and path-scoped-restores the project planning docs (PROJECT.md, STATE.md,
ROADMAP.md, REQUIREMENTS.md, config.json) to their pre-milestone snapshot, committing the
reversion. Ad-hoc-only — refuses on spec/phase-driven milestones with guidance. Destructive;
requires confirmation. Removes local worktrees/branches only — warns (does not delete) if a
milestone branch was pushed to a remote. Lifecycle cleanup removes the snapshot base ref and
ad-hoc config keys.

Usage: `/dgs:abandon-milestone` (interactive; requires confirmation)

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
| `/dgs:code-review [scope\|PR URL]` | On-demand multi-pass code review — auto-fix in local mode, comment-only on PRs | Before complete-quick/complete-milestone, four-eyes PR review, after manual/skip-DGS edits |
| `/dgs:security-review [scope\|PR URL]` | Threat-model-oriented review — reads the diff as new attack surface (trust boundaries, newly-reachable sinks, secrets); reports and routes, never auto-fixes | After code-review settles the diff, when the change plausibly moved attack surface — auth, network, deserialization, secrets |
| `/dgs:adversarial-review [scope]` | Claim-refutation review — refuter agents execute code to refute done-ness claims; verdicts with evidence, no auto-fix | Final trust gate after audit-phase, code-review and security-review, before complete-quick/complete-milestone |
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

Completion honors `git.completion_mode`: in `pr` mode, `/dgs:complete-quick` opens a PR per touched repo instead of merging locally, and the quick parks at `pr_open` until the PR merges (reap with a re-run or `dgs-tools reap-quicks`). See [Completion Modes: Merge vs PR](GIT-WORKFLOW.md#completion-modes-merge-vs-pr).

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

**`/dgs:code-review [staged|unstaged|branch|<file>|<PR URL>] [--repo <name>]`**
On-demand, human-invoked code review at lifecycle boundaries. Distinct from (a) the automatic per-plan codereview gate inside `execute-phase`, which reviews plan-executor output non-interactively, and (b) `/dgs:diff-report`, which summarizes a diff without reviewing it.

- Default scope is context-aware: `branch` when a context is focused or the tree is clean; `staged` otherwise
- **Local mode** (staged/unstaged/branch/file): 3 passes, 12+ parallel agent reviews (Correctness & Security, Standards & Patterns, Simplification, Comprehension Gate, Symmetry & Sweep with cross-reference patterns + per-stack footgun checklists, Failure-Trace, plus optional Deployment & Infrastructure when `.claude/codereview-context.md` exists). Aggressively auto-fixes CRITICAL/HIGH/MEDIUM findings, guarded by a consumer-dependent do-not-autofix list; fixes are test-verified before committing. On `branch` scope, fixes land as one distinct `fix(review):` commit (clean tree in, clean tree out); on staged/unstaged/file scopes, fixes stay uncommitted in the working tree so your next DGS commit carries them
- **PR mode** (GitHub PR URL): review-only — never edits files. Resolves stale review threads, posts ONE consolidated comment, stops. Requires authenticated `gh`; resolves the registered repo and pins `gh` calls with `-R`; fork-safe (`pull/<n>/head` fetching)
- `--repo <name>`: pick a registered repo explicitly
- Outputs: writes `CODE-REVIEW-<timestamp>.md` at the planning root and logs a Code Reviews row + Last activity in STATE.md
- When to run: before `/dgs:complete-quick` (`branch` — whole-quick review including `/dgs:fast` inline commits the per-plan gate never saw); before `/dgs:complete-milestone` (`branch` in the milestone worktree — cross-plan integration issues); as the four-eyes reviewer on a DGS-opened PR (PR URL — second contributor posts a consolidated comment, never edits the author's branch); after manual/skip-DGS edits (`staged`/`unstaged`); external contributions (PR URL)

Usage: `/dgs:code-review` (context-aware default)
Usage: `/dgs:code-review branch` (review all commits on this branch vs base)
Usage: `/dgs:code-review src/lib/foo.cjs` (review one file)
Usage: `/dgs:code-review https://github.com/owner/repo/pull/123` (PR review mode)
Usage: `/dgs:code-review branch --repo my-api` (pick a registered repo explicitly)

**`/dgs:security-review [staged|unstaged|branch|<PR URL>] [--repo <name>] [--base <branch>]`**
Change-scoped security review — the security lens in the gate chain. Where `/dgs:code-review` reads the diff inward for line-level correctness, this reads it outward as *new attack surface*: what does the change let an attacker reach that they could not reach before?

- Stance: a mandatory, written-out reasoning pass — boundary map → reachability delta → sink trace — precedes any finding. Every finding cites the observation it came from and names the entry point, what an untrusted caller can now reach, and the sink it lands in. A candidate that cannot fill all three parts is not a lens finding
- "No attack-surface change" is a first-class, correct outcome — the command deliberately keeps its noise floor low rather than padding with generic line-level nags
- Read-only by construction: the security subagent is granted only `Read`/`Grep`/`Glob`, and a source-change guard proves nothing but the report and STATE.md changed
- Never auto-fixes: findings are presented as a routing table (`/dgs:quick` for contained, a gap phase or `/dgs:add-todo` for systemic) and left for you to action. Non-blocking by convention — it never hard-blocks a completion
- A fail-closed secret scan refuses to produce a report containing a raw secret, with no bypass
- Outputs: writes `SECURITY-REVIEW-<timestamp>.md` at the planning root and logs a Security Reviews row + Last activity in STATE.md
- Two-axis severity: every finding carries `severity` — impact *if the sink is reached*, ordered `info < low < medium < high < critical` — and, independently, `reachability_confidence` (`confirmed` | `probable` | `unproven`), i.e. whether the untrusted path was actually demonstrated from the diff. The axes are deliberately separate: a catastrophic-but-unproven sink keeps its high `severity` and is flagged `reachability_confidence: unproven` rather than being quietly downgraded — so you triage on impact and chase proof as a distinct step
- The lens pins `model: opus`, so the same diff yields the same verdicts regardless of which model you're running the command from (recall and severity anchoring were measured model-sensitive)
- Default scope is `branch` (settled, post-fix state) — unlike code-review's work-in-progress default

Usage: `/dgs:security-review` (default: branch scope)
Usage: `/dgs:security-review staged` (review what's staged)
Usage: `/dgs:security-review branch --repo my-api` (pick a registered repo explicitly)
Usage: `/dgs:security-review branch --base develop` (override the base branch)
Usage: `/dgs:security-review https://github.com/owner/repo/pull/123` (review a PR, fork-safe and read-only)

**`/dgs:adversarial-review [<phase>|<quick-slug>|milestone] [--repo <name>]`**
Claim-refutation review — the final trust gate. Distinct from `audit-phase` (goal-backward structural: does what the plan promised exist and wire up), `/dgs:code-review` (diff-inward line-level: is the code correct/clean/to-standard), and `/dgs:security-review` (trust-boundary-outward: what new attack surface does the diff introduce) — this asks whether the CLAIM that something works is actually true, by executing code rather than reading it.

- Claim sources: phase SUMMARYs, plan `must_haves`/UAT acceptance criteria, VERIFICATION.md conclusions, and commit messages (plus a named extension point for the assumption ledger's `ASSUMED:` entries, when present)
- Dispatches parallel Bash-capable refuter agents rooted at `REVIEW_REPO` with the mandate "run it, don't read it" — execute the test, curl the endpoint, query the table, invoke the CLI with the claimed input
- Verdicts: CONFIRMED (executed evidence), REFUTED (executed evidence of failure, re-verified by the orchestrator before being accepted), UNVERIFIABLE (no executable test, or a non-reproducing refutation)
- No auto-fix: REFUTED claims are routed to `/dgs:quick <description>` (contained) or a gap phase (`/dgs:plan-phase --gaps` / `/dgs:add-todo`, systemic) — never fixed inline
- Outputs: writes `ADVERSARIAL-REVIEW-<timestamp>.md` at the planning root and logs an Adversarial Reviews row + Last activity in STATE.md
- `--repo <name>`: pick a registered repo explicitly

Usage: `/dgs:adversarial-review` (default: most recently completed phase)
Usage: `/dgs:adversarial-review 04` (a specific phase)
Usage: `/dgs:adversarial-review milestone` (all completed phases in the milestone)
Usage: `/dgs:adversarial-review 260704-btz` (a quick slug)
Usage: `/dgs:adversarial-review 04 --repo my-api` (pick a registered repo explicitly)

### Testing & Dependency Scanning

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:package-scan` | Scan registered repos + product root for dependency vulnerabilities and licence issues | Before release, after dependency changes, or on demand |

#### Testing & Dependency Scanning Details

**`/dgs:package-scan [flags]`**

Standalone dependency vulnerability + licence scanner that runs across every repo in `REPOS.md` plus the product root.

- **Tool cascade (default `auto`):** Snyk (if token configured) → OSV-Scanner (if on PATH) → ecosystem-native tool per repo (`npm audit`, `pip-audit`, `govulncheck`, `bundler-audit`). Java has no standard native tool — scans fall back to Snyk or OSV.
- **Ecosystems detected via manifest files:** Node.js (`package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`), Python (`requirements.txt`, `Pipfile.lock`, `poetry.lock`, `pyproject.toml`, `setup.py`), Go (`go.mod`, `go.sum`), Ruby (`Gemfile.lock`, `Gemfile`), Java (`pom.xml`, `build.gradle`, `build.gradle.kts`).
- **Monorepo awareness:** npm/pnpm/Yarn workspaces, Maven multi-module, and Go workspaces (`go.work`) are expanded into per-workspace targets with `manifest_path` attribution in each finding. Gradle sub-projects and Python src-layout monorepos are deferred (PKG-41, PKG-42 — treated as single-module).
- **Report placement:** `{phase-dir}/{phase}-PACKAGE-SCAN.md` when an active phase is set, else `milestones/v{X}.{Y}-PACKAGE-SCAN.md` when an active milestone, else timestamped `PACKAGE-SCAN-{YYYY-MM-DD-HHmm}.md` at the product root.
- **Report format:** YAML frontmatter carries `type`, `date`, `tool`, `repos_scanned`, severity counts, `duration`, and a `findings:` array in canonical shape. Body renders a per-repo × severity summary table, per-severity sections, and a Licence Compliance section when Snyk is the selected tool.
- **Plan provenance:** each finding carries optional `introduced_in_commit` (7-char SHA) and `introduced_in_plan` (DGS plan-id) resolved via `git log -S <package> -- <manifest>` on the manifest file.
- **Exit code:** 0 when the scan completes (findings are not errors). Non-zero is reserved for tool unavailability or infrastructure failure.

**Flags:**

| Flag | Effect |
|------|--------|
| `--threshold critical\|high\|medium\|low` | Filter findings at or above the severity. Default from `testing.packages.severity_threshold` config. |
| `--repo <name>` | Restrict scan to one registered repo (errors with a valid-repo list if the name is unknown). |
| `--json` | Emit machine-readable JSON to stdout in addition to writing the markdown report. |
| `--include-dev-deps` / `--no-include-dev-deps` | Include or exclude devDependencies. Default from `testing.packages.include_dev_dependencies` config. |

**Configuration:** see [Configuration Reference](CONFIGURATION-GUIDE.md#testing--package-scanning) for `testing.packages.*` keys (tool selection, severity threshold, Snyk token placement, timeout).

**Related files in the DGS install:**
- `deliver-great-systems/references/package-scan-config.md` — full user-facing config reference + tool installation instructions.
- `deliver-great-systems/workflows/package-scan.md` — command workflow definition.
- `deliver-great-systems/templates/package-scan-report.md` — documentation-only example of the report format.
- `deliver-great-systems/skills/dgs-tests/package-scan.md` — test-gate plugin skill file (inert, forward-compatible with the skills engine).

Usage:
```
/dgs:package-scan                              # scan everything with cascade default
/dgs:package-scan --threshold high             # high/critical only
/dgs:package-scan --repo api                   # single repo
/dgs:package-scan --json                       # emit JSON output
/dgs:package-scan --threshold critical --repo worker --no-include-dev-deps
```

### Artifact Graph

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:query "<question>"` | Natural-language question answered as a read-only SELECT over the SQLite artifact index | "Which ideas are pending?", tracing idea → spec → milestone → phases |
| `/dgs:integrity` | Report dangling links, orphaned artifacts, and broken milestone references (read-only) | Before release, in CI / a pre-push hook, or on demand |

#### Artifact Graph Details

**`/dgs:query "<question>" [--sql] [--limit N]`**
Ask a natural-language question of the artifact graph. Read-only NL→SQL over the SQLite index.

- Translation runs entirely in your local session — no new external model or telemetry
- Produces exactly one read-only `SELECT`, executed through an engine-level safety guard: SELECT + safe-PRAGMA whitelist, injected `LIMIT`, a 5s statement interrupt, and parameter binding. A non-SELECT/DDL query is rejected fail-closed with the offending SQL shown
- Results render in the DGS list/progress style with an always-on truncation footer; supports conversational follow-up refinement
- Answers reverse-link and full-chain traceability questions (what references idea N; idea → spec → milestone → phases)
- Backed by `dgs-tools query {schema,sql,trace,reverse-links}` — the command never opens the DB directly
- `--sql`: show the generated query. `--limit N`: cap the row count
- Config gate: when `query.enabled` is `false` the SQL is generated and shown but not auto-executed (headless/CLI-only opt-out — see [Configuration Reference](CONFIGURATION-GUIDE.md#query))
- Degrades gracefully: when the SQLite index is absent it prints an info line instead of failing

Usage: `/dgs:query "which ideas are still pending?"`
Usage: `/dgs:query "phases in milestone v28.0" --sql` (show the generated SQL)
Usage: `/dgs:query "specs referencing idea 12" --limit 20`

**`/dgs:integrity [--quiet]`**
Report artifact-graph integrity problems, grouped by class, with per-class and total counts. A clean graph prints `✓ no integrity issues`.

- Classes: dangling `id-ref`/link targets, orphaned artifacts, broken `milestone` references
- Strictly read-only; target existence is decided through the virtual-aware resolver, so virtual targets (`quick`, `fast`) are never false-flagged while a `milestone` that resolves to nothing IS reported
- Exits nonzero when any finding exists (CI / pre-push-hook ready). Deliberately NOT auto-wired into DGS's own commit gate in v28.0 (deferred)
- `--quiet`: summary counts only
- When the SQLite index is absent it degrades to a filesystem fallback (dangling + broken-milestone)

Usage: `/dgs:integrity` (full grouped report)
Usage: `/dgs:integrity --quiet` (summary counts only)

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
- Creates a structured research document at `docs/ideas/{slug}-research.md` with frontmatter (type, idea_id, idea_title, date, repos_analysed)
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

If your project has codebase maps (`codebase/`) or product docs (`docs/product/`), DGS loads them as context during conversion. This means the Implementation Notes section references real modules and patterns from your codebase rather than generic placeholders.

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

### Threads

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dgs:add-thread [title]` | Create a thread — an organizing topic above ideas | Starting a line of work that will generate multiple ideas/todos and decisions over time |
| `/dgs:list-threads` | Thread index: status + live child idea/todo roll-up per thread | Check what threads are open, dormant, or closed |
| `/dgs:resume-thread [thread-id]` | Reload a thread's goal, decisions ledger, next steps, and open todos | Picking a thread back up in a new session |
| `/dgs:promote-idea [idea-id]` | Promote an existing idea into the thread it anchors | An idea has grown into an ongoing topic that will spawn more ideas/todos |
| `/dgs:backfill-threads` | Adopt threads onto an existing idea backlog via a review-gated dry-run sweep | Retrofitting threads onto ideas captured before threads existed |

#### Threads Details

**`/dgs:add-thread [title] [--id thread-slug] [--projects a,b]`**
Create a single thread directly, parallel to `/dgs:add-idea`. Mints the thread doc, seeds an empty Goal + Next Steps, and records a create Log entry — all in ONE commit. Create-only: this command runs no dedup pass itself — the create-time similarity/anti-fragmentation guard lives in the engine (see the Similarity guard note in [USER-GUIDE.md](USER-GUIDE.md#threads)).

- `--id thread-slug`: explicit id (otherwise auto-derived from the title)
- `--projects a,b`: tag the thread with project scope
- Conversational title resolution when no title is given — never forces you to type the verb

Usage: `/dgs:add-thread` (prompts for a title)
Usage: `/dgs:add-thread "Payments v2 rework"` (title-only create; id auto-derived from the title)

**`/dgs:list-threads [--status open|dormant|closed] [--projects a,b]`**
A THREAD INDEX — rows are the threads themselves, each with its status and a live roll-up of child ideas + open todos (`{idea_count} ideas / {todo_count} todos`), in `updated` desc order. Distinct from `list-ideas --by-thread` (an ideas view grouped under thread headers).

Usage: `/dgs:list-threads` (all threads)
Usage: `/dgs:list-threads --status open` (only open threads)

**`/dgs:resume-thread [thread-id] [--reopen]`**
Load a thread's goal, decisions ledger, next steps, and open todos into the session. A dormant thread flips to open automatically (audited). A closed thread NEVER auto-flips — it prompts before reopening; a non-interactive invocation displays without flipping, and `--reopen` performs the audited flip.

Usage: `/dgs:resume-thread payments-v2-rework`
Usage: `/dgs:resume-thread payments-v2-rework --reopen` (reopen a closed thread)

**`/dgs:promote-idea [idea-id] [--id thread-slug] [--relink]`**
Promote an existing idea into the anchoring thread it represents. Creates the thread and stamps the source idea's single `links.thread` edge in ONE atomic commit. The idea is RETAINED additively as the thread's anchor child — its status is never changed, and it keeps its full lifecycle (idea → spec → phase is still possible).

- `--id thread-slug`: explicit thread id (otherwise auto-derived from the idea's title)
- `--relink`: required to move an idea that is already linked to a different thread (never auto-passed)

Usage: `/dgs:promote-idea 42` (promote idea #42 into a new anchoring thread)

**`/dgs:backfill-threads [--projects a,b]`**
Adopt threads on an EXISTING pending-ideas backlog via a review-gated, revertible dry-run classification sweep. Runs a deterministic dry-run classification, renders cluster-parent proposals conversationally (proposed title, one-line goal, child count + titles, confidence, ambiguity flags), shows a separate "unsure" bucket and a reported-defects list, and lets you accept/reject in natural language with bulk-accept and edit-at-review. Claude then AI-synthesizes each accepted thread's Goal + Context and attributed Decisions Ledger, applies via the engine `thread sweep-apply`, and surfaces the one-command revert. Conversational review is the primary write surface — the raw `dgs-tools thread sweep` JSON stays available for non-interactive/automation consumers.

Usage: `/dgs:backfill-threads` (dry-run sweep + conversational review)
Usage: `/dgs:backfill-threads --projects api,web` (scope the sweep to specific projects)

**`dgs-tools thread <verb>`**
The full engine roster backing the five slash commands above (mirrors how `dgs-tools query …` backs `/dgs:query` in [Artifact Graph Details](#artifact-graph-details) and `dgs-tools reap-quicks` backs the reap sweep in [PR Completion & reap-quicks](#pr-completion--reap-quicks)):

`create`, `list`, `append-note`, `append-context`, `append-decision`, `append-retraction`, `set-status`, `close`, `compact`, `link`, `promote`, `resume`, `sweep`, `sweep-apply`, `sweep-revert`

- `list` and `sweep` are read-only — no git-identity gate, nothing written. Every other verb gates identity.
- `append-note` / `append-context` / `append-decision` route an attributed, ISO-8601-UTC-timestamped entry to the Log / Context / Decisions Ledger section respectively; `append-retraction --target <addr> --reason <text>` appends a retraction entry rather than editing history.
- `set-status <id> <open|dormant|closed>` and `close <id>` transition lifecycle.
- `compact [--goal <text>] [--context <text>] [--next-steps <text>] [--force]` rewrites ONLY the steward-prose sections it's given — see the anatomy note in [USER-GUIDE.md](USER-GUIDE.md#threads): the Decisions Ledger and Log are never rewritten by `compact`, no matter which of the three flags are passed.
- `link <id> <child-ref> [--type idea|todo] [--relink]`: stamp an existing idea or todo's single forward `links.thread` edge (default `--type idea`); `--relink` is required to move a child that already points at a different thread (cardinality-one — a child owns at most one thread edge).
- `promote --idea <id> [--id thread-slug] [--relink]`: the engine call behind `/dgs:promote-idea`.
- `sweep [--projects a,b]`: dry-run classify, creates nothing — the engine call behind `/dgs:backfill-threads`'s dry-run pass.
- `sweep-apply --accept <file>`: apply a reviewed accept-spec JSON file.
- `sweep-revert --run-id <id> [--force]`: revert a sweep-apply run by its run id; safe-by-default, `--force` overrides drift refusals.

Thread `id` is an immutable, auto-derived SLUG (not an integer) — derived from the title via `slugifyThreadId` when `--id` is omitted. Lifecycle is `open` / `dormant` / `closed`; appends to a closed thread are rejected (the file is left untouched) with a reopen hint (`reopen with: dgs-tools thread set-status <id> open`). Roll-up counts (`idea_count`/`todo_count`) are always live reverse-index queries — never text stored in the thread file itself.

**Lineage flags.** Sibling commands carry thread-aware engine flags (`dgs-tools` layer): `ideas create --from-thread <id>` and `todo create --from-thread <id>` (mint a new idea/todo pre-linked to a thread, stamping its `links.thread` edge in the SAME create commit), `ideas list --by-thread` and `list-todos --by-thread` (group results under thread headers), and `thread link --relink` (the cardinality-one override described above — moving a child to a different thread always requires it, never silently overwritten).

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
- **`/dgs:write-spec`** and **`/dgs:plan-phase`** load all markdown files from `docs/product/`
- **`/dgs:new-milestone`** loads `ARCHITECTURE.md` and `PRODUCT-SUMMARY.md` from `docs/product/` (if they exist) to inform research and roadmap creation

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
| `/dgs:init-product` | Initialize planning repo structure with recommended defaults | Setting up multi-repo management |
| `/dgs:add-repo [name]` | Register a sibling repo | Adding a new repo to the project |
| `/dgs:remove-repo [name]` | Unregister a repo from REPOS.md | Removing a repo from the product |
| `/dgs:overlap-check` | Show repos touched by multiple active projects | Detect potential cross-project conflicts |
| `/dgs:health` | Validate repo reachability and config | After setup or when repos seem unreachable |

### Per-Console Context

The active context (a milestone or a standalone quick) is resolved **per console** with precedence `--context` flag > **session binding** (`CLAUDE_CODE_SESSION_ID` → `execution.console_bindings`) > `DGS_CONTEXT` env var > config default (`execution.active_context`) > none. Inside a Claude Code session the session binding is the default path (no shell setup); `DGS_CONTEXT` is the raw-shell path. See [Per-Console Context](USER-GUIDE.md#per-console-context) in the User Guide for both workflows.

**Session binding (`CLAUDE_CODE_SESSION_ID` → `execution.console_bindings`).** In a Claude Code session, `switch-context <slug>` and quick-create bind the **current session** by writing `config.local.json` `execution.console_bindings[CLAUDE_CODE_SESSION_ID] = <slug>` instead of clobbering the shared `execution.active_context` — so parallel sessions on one project each keep their own focus. `complete-quick` / `abandon-quick` / `complete-milestone` drop every binding pointing at the finished slug. No `.zshrc` edit and no relaunch are needed.

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `dgs-tools switch-context <slug>` | In a Claude Code session: bind THIS session (`execution.console_bindings[CLAUDE_CODE_SESSION_ID]`). In a raw shell with no session id: repoint the shared config default | Focus the current console on a milestone or quick for parallel work |
| `dgs-tools switch-context <slug> --print` | Raw-shell path: emit `export DGS_CONTEXT=<slug>` on stdout (read-only — writes no config). Eval it to bind a plain terminal: `eval "$(dgs-tools switch-context <slug> --print)"` | Bind a non-Claude-Code terminal to a context |
| `dgs-tools context` (or `context show`) | Print the resolved per-console context + source as `<slug> (session\|flag\|env\|default)`; a session-bound console shows `<slug> (session)` | Check what this console is bound to |

**`--context <slug>` flag.** A one-shot per-command override accepted on the context-sensitive command set: `fast`, `quick`, `complete-quick`, `abandon-quick`, and `complete-milestone`. It binds the command's context at the FLAG precedence level (flag > session binding > `DGS_CONTEXT` env > config default) and never overwrites the inherited session binding or `DGS_CONTEXT` — a stale `--context` (no live worktree) warns and falls through to the next level.

**Per-context executor lock (`execution.executing`).** While `/dgs:execute-phase` runs a wave, it records a lock in `config.local.json` `execution.executing`, keyed by **context slug** (`{ "<slug>": { started_at, session_id } }`). `switch-context` refuses to flip focus into or out of a context that is mid-execution (drift guard), but a lock on one context never blocks an independent console working a different context. Each entry honours a 6h-stale escape and the `--force` override; `--print` (read-only) is exempt.

**`unset DGS_CONTEXT` advisory.** When `complete-quick` or `abandon-quick` finalizes a quick the current console is bound to (`DGS_CONTEXT` equals that slug), the tool emits an `unset DGS_CONTEXT` line (stderr) and an `unbind` field in its JSON, so a raw shell can clear a binding that now points at a removed worktree. (Session bindings are cleared automatically.)

**Safety guards (Phase 9).** `complete-quick` refuses a milestone-bound console and `complete-milestone` refuses a quick-bound console (the actionable message names the bound slug and the correct command). An unbound console prints a one-time `unbound — defaulting to <milestone>` notice before context-sensitive commands. The standalone-quick cap (3) counts quick worktrees only — the milestone context never consumes a slot.

### PR Completion & `reap-quicks`

**`git.completion_mode`.** `complete-quick` and `complete-milestone` honor the `git.completion_mode` config key: `merge` (the default — an absent key means `merge`) or `pr`. In `pr` mode, completion rebases, pushes the branch with `--force-with-lease`, and opens (or updates) a GitHub pull request per touched repo instead of merging locally; the quick or milestone parks at `pr_open` until its PRs merge, then a re-run reaps it. The GitHub CLI (`gh`) is a hard dependency **only** in `pr` mode (and for fast-PR) — the default `merge` path never invokes it. Full model: [Completion Modes: Merge vs PR](GIT-WORKFLOW.md#completion-modes-merge-vs-pr).

**`dgs-tools reap-quicks`**
Sweep every live quick and reap the ones whose PRs have merged. For each quick at `pr_open` with a stored PR record, the sweep checks merge state via `gh` — every repo of a multi-repo quick must be merged; a partial merge never reaps. Merged quicks are reaped (base branch pulled, worktree and branch removed, console bindings dropped). Everything else is skipped with a reason, and a `gh` outage on one quick fails that quick only — the sweep continues. Output is a per-quick summary with `reaped` / `skipped` / `failed` counts.

| Flag | Effect |
|------|--------|
| `--merged` | Assert the merge and reap **without** `gh` — the probe is bypassed entirely. The escape hatch when `gh` is down or unauthenticated and you know the PR merged. |
| `--slug <slug>` | Narrow the sweep to a single named quick. |
| `--force-reap` | Override the post-merge-work guard (commits made in the worktree after the last pushed PR head otherwise block the reap). |

A quick whose PR was closed **without** merging is never swept — it is skipped with guidance to run `complete-quick <slug> --confirm-cleanup` instead.

Usage: `dgs-tools reap-quicks` (sweep all merged quicks)
Usage: `dgs-tools reap-quicks --merged --slug 260702-abc` (assert one quick merged, no gh)
