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

## Threads: the committed multi-user convergence primitive

Unlike the deprecated branching model above, **threads** (`threads/<id>.md`) are DGS's first genuinely-current multi-user capability — a committed, git-native primitive for multiple developers appending to the same shared record without manual conflict surgery.

Every thread file carries `threads/*.md merge=union` in `.gitattributes`, so git's own `union` merge driver concatenates concurrent appends from different clones instead of conflicting on them. On top of that, thread writes go through a `commit → push → pull --rebase → retry-once` sync wrapper: a write commits locally, pushes, and — if the push is rejected as non-fast-forward — rebases onto the fetched remote and retries exactly once before giving up.

Convergence is deterministic, not best-effort. Every appended entry (Context, Decisions Ledger, Log) carries a canonical address of `(timestamp, author email, SHA-256 content hash)`; after a union merge, entries are sorted by that address and exact-duplicate triples are dropped — so two clones that raced to append the same entry converge on one copy, in the same order, everywhere, with zero manual conflict resolution.

Prose consolidation is the one operation this scheme can't auto-merge safely, so it is deliberately NOT part of the append path: `thread compact` (rewriting Goal/Context/Next Steps into clean prose) is a guard-railed, coordinate-first, solo-moment operation — it requires a clean tree, fetches first, and refuses to run if the remote has un-pulled changes to that thread file (override with `--force`). The Decisions Ledger and Log are never touched by `compact`, so the append-only convergence guarantee above never has to account for a concurrent prose rewrite.

This same deterministic ordering is what keeps `/dgs:list-threads` and the dashboard's thread listing stable across clones — two developers on two machines see threads in the same order, with the same live child roll-ups, regardless of which of them pushed first.

Minimum git version: **2.23** (required for the `merge=union` driver behavior this relies on).

See [Threads](USER-GUIDE.md#threads) in the User Guide for the full document anatomy and lifecycle.

---

*Deprecated as of v25.3 — this file is retained as a redirect stub for inbound-link stability. The Threads section above is the exception: it documents a currently-shipped (v33.0) capability, not the deprecated branching model.*
