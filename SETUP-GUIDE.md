# Setup Guide

> See also: [USER-GUIDE.md](USER-GUIDE.md) for workflow overview and usage examples.

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
/dgs:new-project                     # establish project identity (PROJECT.md)
/dgs:new-milestone                   # research, requirements, roadmap for first milestone
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

### REPOS.md Setup Field

The optional `setup` field in REPOS.md specifies a command to run when DGS creates a worktree for that repo. This handles dependency installation, build steps, or any environment preparation.

```markdown
| Name | Path | Setup |
|------|------|-------|
| api | ../api | npm install |
| web | ../web | ./scripts/setup-worktree.sh |
| mono | ../mono | pnpm install --frozen-lockfile |
```

The setup command receives:

- **$1** — the slug (e.g., `v19`, `fix-auth-bug`)
- **$2** — the absolute path to the worktree directory
- **cwd** — set to the worktree directory
- **Timeout** — 5 minutes

If setup fails, the worktree remains in valid git state. Fix the issue and re-run:

```
dgs-tools worktrees setup {slug}
```

For monorepos: the worktree is at the repo level. Your setup script handles internal topology (workspace installation, shared package builds). DGS has no monorepo-specific logic.

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
      ARCHITECTURE.md               #     Target architecture (used by new-milestone)
      PRODUCT-SUMMARY.md            #     Product summary (used by new-milestone)
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
    research/                       #   Domain research from /dgs:new-milestone
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
