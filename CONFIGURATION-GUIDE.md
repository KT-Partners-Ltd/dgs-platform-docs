# Configuration Reference

> See also: [USER-GUIDE.md](USER-GUIDE.md) for workflow overview and usage examples.

DGS stores product-level settings in `dgs.config.json` (in your planning root). This file is shared across all projects in the product. Configure during `/dgs:init-product`, and update later with `/dgs:settings`.

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
    "base_branch": "main"
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

### Git Settings

| Setting | Options | Default | What it Controls |
|---------|---------|---------|------------------|
| `git.base_branch` | Branch name string | `main` | Integration target for all merge, rebase, and push operations |

DGS uses git worktrees for all isolation. Each milestone and product-level quick task gets its own worktree on a dedicated branch. There is no `branching_strategy` config — the worktree model is always active. See [Quick Workflows](MILESTONE-JOBS-GUIDE.md#quick-workflows) for how worktrees are managed during quick tasks, and the milestone lifecycle section for milestone worktrees.

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
