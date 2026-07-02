# Configuration Reference

> See also: [USER-GUIDE.md](USER-GUIDE.md) for workflow overview and usage examples.

DGS stores product-level settings in `dgs.config.json` (in your planning root). This file is shared across all projects in the product. Recommended defaults are applied automatically by `/dgs:init-product`. Update any setting later with `/dgs:settings`.

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
    "codereview": false,
    "four_eyes": "off"
  },
  "testing": {
    "packages": {
      "tool": "auto",
      "severity_threshold": "low",
      "include_dev_dependencies": true,
      "timeout_seconds": 300
    }
  },
  "git": {
    "base_branch": "main",
    "completion_mode": "merge",
    "sync_push": "off",
    "sync_pull": "off"
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
| `planning.commit_docs` | `true`, `false` | `true` | Whether planning files (STATE.md, ROADMAP.md, phases/, etc.) are committed to git |
| `planning.search_gitignored` | `true`, `false` | `false` | Add `--no-ignore` to broad searches to include planning files |

> **Note:** If the planning repo is gitignored from a source repo, `commit_docs` is automatically `false` regardless of the config value.

### Workflow Toggles

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `workflow.research` | `true`, `false` | `true` | Domain investigation before planning |
| `workflow.plan_check` | `true`, `false` | `true` | Plan verification loop (up to 3 iterations) |
| `workflow.verifier` | `true`, `false` | `true` | Post-execution verification against phase goals |
| `workflow.nyquist_validation` | `true`, `false` | `true` | Validation architecture research during plan-phase; 8th plan-check dimension |
| `workflow.codereview` | `true`, `false` | `false` | 3-pass, 9-agent multi-agent code review after each plan execution. Auto-fixes low-risk issues in a separate commit. |
| `workflow.four_eyes` | `off`, `warn`, `enforce` | `off` | Completion governance: checks whether the user completing a milestone or quick task contributed to the work. `warn` proceeds with audit log; `enforce` blocks unless `--force`. See [Multi-User Guide](MULTI_USER_APPROACH_GUIDE.md#completion-governance-four-eyes). |

Disable these to speed up phases in familiar domains or when conserving tokens.

### Testing & Package Scanning

Configuration for `/dgs:package-scan` (introduced in v23.1). See [`references/package-scan-config.md`](../deliver-great-systems/references/package-scan-config.md) for the full user-facing reference, tool installation steps, and report-format details.

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `testing.packages.tool` | `auto`, `snyk`, `osv`, `native` | `auto` | Tool-selection strategy. `auto` cascades Snyk → OSV-Scanner → ecosystem-native tool based on availability. A pinned tool that is not installed causes a fast-fail with an install hint (no silent fallback). |
| `testing.packages.severity_threshold` | `critical`, `high`, `medium`, `low` | `low` | Minimum severity included in the report. Also the default for the `--threshold` CLI flag. |
| `testing.packages.include_dev_dependencies` | `true`, `false` | `true` | Whether devDependencies are scanned. Maps to tool-specific argv (`--production` for Snyk, `--omit=dev` for npm audit, etc.). Also the default for the `--include-dev-deps` / `--no-include-dev-deps` CLI flag. |
| `testing.packages.timeout_seconds` | Integer in `[10, 3600]` | `300` | Per-scan-invocation timeout (seconds). Applies to each spawned tool subprocess. |

**Local-only key (gitignored, stored in `config.local.json`):**

| Setting | Format | What it Controls |
|---------|--------|------------------|
| `testing.packages.snyk_token` | Snyk API token string | Snyk authentication. MUST be set via `dgs-tools config-local-set testing.packages.snyk_token <token>` — `config-set` rejects this key with guidance to use the local path. Alternatively set `SNYK_TOKEN` in your shell env or run `snyk auth` (DGS honours all three sources in this priority: `config.local.json` → `SNYK_TOKEN` → `snyk config get api`). |

Set via (shared keys):
```
dgs-tools config-set testing.packages.tool snyk
dgs-tools config-set testing.packages.severity_threshold high
dgs-tools config-set testing.packages.include_dev_dependencies false
dgs-tools config-set testing.packages.timeout_seconds 600
```

Set via (local-only Snyk token):
```
dgs-tools config-local-set testing.packages.snyk_token <your-token>
```

**Tool installation** — see [`references/package-scan-config.md`](../deliver-great-systems/references/package-scan-config.md#tool-installation) for install commands per tool (Snyk, OSV-Scanner, pip-audit, govulncheck, bundler-audit).

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

### Git Settings

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `git.base_branch` | Branch name string | `main` | Integration target for all merge, rebase, and push operations |
| `git.completion_mode` | `merge`, `pr` | `merge` | How completed quicks and milestones integrate: `merge` rebases and merges locally; `pr` rebases, pushes with `--force-with-lease`, and opens a GitHub pull request per touched repo. The GitHub CLI (`gh`) is required **only** in `pr` mode. See [Completion Modes: Merge vs PR](GIT-WORKFLOW.md#completion-modes-merge-vs-pr). |
| `git.sync_push` | `off`, `prompt`, `auto` | `off` | Whether workflows push planning-repo commits to the remote at their built-in cadence points |
| `git.sync_pull` | `off`, `prompt`, `auto` | `off` | Whether workflows pull shared planning state from the remote at their built-in cadence points |
| `git.sync` | `off`, `prompt`, `auto` | — | Convenience shorthand: setting it writes the same value to **both** `git.sync_push` and `git.sync_pull` in one call. It is not a third stored setting. |

DGS uses git worktrees for all isolation. Each milestone and product-level quick task gets its own worktree on a dedicated branch. There is no branch-strategy setting to choose — the worktree model is always active. See [Quick Workflows](MILESTONE-JOBS-GUIDE.md#quick-workflows) for how worktrees are managed during quick tasks, and the milestone lifecycle section for milestone worktrees.

**Sync cadence is fixed, not configurable.** Which workflows sync — and whether they pull, push, or both — is a built-in classification baked into the sync engine (every DGS workflow is classified as pull+push, push-only, pull-only, or no-sync; e.g. `execute-phase` and `plan-phase` pull and push, `add-idea` pushes only, `progress` pulls only, `help` never syncs). The `git.sync_push` / `git.sync_pull` settings control only the *mode* at those built-in cadence points: `off` skips the sync, `prompt` asks first, `auto` syncs silently. You cannot change which workflows sync, only how the sync behaves when a workflow reaches its cadence point.

### Local Execution State (config.local.json)

Alongside the shared, git-tracked config file, DGS keeps per-machine state in `config.local.json` (gitignored, next to the shared config). The entire `execution.*` namespace routes here — these keys never appear in the tracked config file:

| Key | Shape | What it Tracks |
|-----|-------|----------------|
| `execution.console_bindings` | `{ <session-id>: <context-slug> }` | Which context each Claude Code session is focused on — how per-console focus persists across commands in parallel sessions. Bound by `/dgs:switch-context` (and on quick creation); dropped when the bound context completes. |
| `execution.executing` | `{ <slug>: { started_at, session_id } }` | The execution lock — which session is currently executing a phase for a given context. Heartbeat-refreshed at wave boundaries; stale after 6h. |
| `execution.active_context` | Context slug string | The default focused context (worktree focus pointer) used when no flag, session binding, or `DGS_CONTEXT` env applies. |
| `execution.fast_pr_map` | `{ <branch>: <pr-record> }` | Fast-PR branch tracking used by the stateless `fast/*` branch self-prune. |

These keys are **system-managed** — DGS reads and writes them under a cross-process lock; you should not normally hand-edit them. They are useful when troubleshooting: a stuck execution lock (e.g. after a crashed run) is released with `dgs-tools execution-lock release <slug> --force` — see [Recovering a Stuck Run](GIT-WORKFLOW.md#recovering-a-stuck-run-the-execution-lock). Per-console focus resolution is described in [Parallel Consoles & Focus](GIT-WORKFLOW.md#parallel-consoles--focus).

### Conflict Resolution

When `/dgs:complete-milestone` rebases and merges a worktree branch back to the base branch, merge conflicts can occur.

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
