# Multi-User Guide for DGS

> **Deprecated.** This guide described an older branching model that predates the current worktree-based git model and is no longer accurate.
>
> For the current git model — worktrees, branch lifecycle, and how work completes — see **[How Git is Used in DGS](GIT-WORKFLOW.md)**, including its concurrency & locking section.

The one section retained below is [Completion Governance (Four-Eyes)](#completion-governance-four-eyes), which remains current. All other content has been removed.

## Completion Governance (Four-Eyes)

The four-eyes principle requires that the person who performs work is not the same person who approves it. DGS supports this for milestone completion and quick task completion through a configurable governance gate.

Enable it with:

```bash
dgs-tools config-set workflow.four_eyes warn   # values: off | warn | enforce
```

| Mode | Behavior |
|------|----------|
| `off` (default) | No contributor check |
| `warn` | Completion proceeds even if the completing user contributed to the work; the override is logged to MILESTONES.md and STATE.md |
| `enforce` | Completion is blocked if the completing user contributed to the work, unless `--force` is used; forced overrides are logged |

When `/dgs:complete-milestone` or `/dgs:quick-complete` runs, DGS compares the completing user against the contributor list recorded for the work and applies the configured mode.

---

*Deprecated as of v25.3 — this file is retained as a redirect stub for inbound-link stability.*
