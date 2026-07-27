# Getting Started

A single, linear walkthrough from zero to your first shipped phase. No branching
decision trees — just follow the steps in order. For deeper reference material,
each step links out to the relevant guide.

---

## 1. Install

```bash
npx @ktpartners/dgs-platform@latest
```

The installer prompts you to choose:
1. **Runtime** — Claude Code, OpenCode, Gemini, or all
2. **Location** — Global (all projects) or local (current project only)

Verify the install worked:

```
/dgs:help
```

If you see the command list, you're ready.

---

## 2. Set Up Your Repos

DGS runs from a **planning repo** — a git repo where DGS stores ALL of its planning
artifacts (config, requirements, roadmaps, ideas, specs, docs) at the repo root —
plus, for most projects, one or more **sibling code repos** living in the same
parent directory. Your source code lives in the sibling repos; the planning repo
holds only planning artifacts.

```
~/projects/
├── my-product/            <- planning repo (run DGS here; config.json, REPOS.md,
│   │                         PROJECTS.md, ideas/, specs/, docs/, projects/<slug>/ at root)
│   └── .planning/.index/  <- rebuildable SQLite index (gitignored)
├── api-service/           <- code repo (sibling)
└── web-app/               <- code repo (sibling)
```

Create the repos and GitHub origins BEFORE running `/dgs:init-product`:

1. Create and enter the planning repo:
   ```bash
   cd ~/projects && mkdir my-product && cd my-product && git init
   ```
   Or, with the GitHub CLI, create the repo and origin together:
   ```bash
   gh repo create my-product --private --source . --remote origin
   ```
2. Create each code repo as a SIBLING of the planning repo (not nested inside it):
   ```bash
   cd .. && mkdir api-service && cd api-service && git init
   ```
3. Give each repo — planning and code — a GitHub `origin`:
   ```bash
   gh repo create api-service --private --source . --remote origin
   ```
   or, if the GitHub repo already exists:
   ```bash
   git remote add origin git@github.com:org/api-service.git
   ```
4. `/dgs:init-product` (next step) must be run from INSIDE the git planning repo —
   it errors if you're not in one. Code repos are registered LATER, with
   `/dgs:add-repo <name>` (using `../` paths).

Single-repo projects can skip step 2 — just create and initialize the one repo and
move on to the next step. See [SETUP-GUIDE.md](SETUP-GUIDE.md) for the full
multi-repo model.

---

## 3. Initialize Your Product

```
/dgs:init-product
```

Run this once, inside your planning repo. It creates the DGS structure at the
planning-repo root — `config.json`, `PROJECTS.md`, `REPOS.md` — plus the `ideas/`,
`specs/`, `docs/`, and `quick/` directories, and scaffolds two stub documents at
`docs/product/PRODUCT-SUMMARY.md` and `docs/product/ARCHITECTURE.md`. It also
creates `.planning/.index/`, a rebuildable, gitignored SQLite index — the only
thing that lives in that hidden directory. Everything else in DGS builds on top
of this one-time setup.

---

## 4. Ground the System

This is the step newcomers skip and later regret. Before you build anything, fill
in the two stub files `/dgs:init-product` just created:

- `docs/product/PRODUCT-SUMMARY.md` — what your product is, who it's for, and what matters
- `docs/product/ARCHITECTURE.md` — your system's real shape: modules, tech choices, security model

**Why this matters:** `/dgs:new-milestone`, `/dgs:write-spec`, and `/dgs:plan-phase`
all READ these two files as context before they research, scope, or plan anything.
An ungrounded system means these agents are guessing at what you're building. A
grounded one means milestone specs and phase plans are accurate from the first draft
— fewer wasted planning cycles, less rework, output that actually fits your product.

Spend five minutes here. It pays for itself on the very first milestone.

Both files are scaffolded with the same template shown below — fill in the HTML
comments, delete them, or leave them as prompts for later.

### `docs/product/PRODUCT-SUMMARY.md`

```markdown
# Product Summary

<!-- Fill this in. DGS agents (/dgs:new-milestone, /dgs:write-spec, /dgs:plan-phase)
     read this file to ground their output in what your product actually is.
     Keep it concise — a page or less. -->

## What This Is

<!-- One or two sentences: what the product does and the problem it solves. -->

## Who It's For

<!-- The primary users and what they need. -->

## Core Value

<!-- The single most important outcome the product delivers. -->

## Key Constraints

<!-- Non-negotiables: compliance, platforms, performance budgets, tech mandates. -->
```

### `docs/product/ARCHITECTURE.md`

```markdown
# Architecture

<!-- Fill this in. DGS agents (/dgs:new-milestone, /dgs:write-spec, /dgs:plan-phase)
     read this file to ground technical decisions in your real system shape.
     Keep it concise and current. -->

## System Overview

<!-- The major moving parts and how a request/data flows through them. -->

## Module Boundaries

<!-- The main modules/services and what each owns. -->

## Tech Choices

<!-- Languages, frameworks, datastores, and why. -->

## Security Model

<!-- Auth, trust boundaries, secret handling, sensitive-data rules. -->
```

> Already have code? Run `/dgs:map-codebase` before this step — it analyzes your
> registered repos and can inform what you write in `ARCHITECTURE.md`.

Grounding isn't limited to the two templates above. If you already have a PRD, an
architecture diagram, or a product one-pager, load it with
`/dgs:add-doc <file> --scope product` — it copies the file into `docs/product/` with
text extraction, and the same grounding agents (`/dgs:new-milestone`, `/dgs:write-spec`,
`/dgs:plan-phase`) read those product-scoped docs as context. So grounding means: fill
the two templates AND/OR attach existing docs via `add-doc`.

---

## 5. Start Your First Project

```
/dgs:new-project
```

`/dgs:new-project` establishes your project's identity through interactive
questioning that starts by naming your project — the name becomes the project's
identity and its `projects/<slug>/` folder. It then asks about goals, constraints,
tech preferences, and edge cases, and creates `PROJECT.md`.

If you already have a product doc or a finalized spec, skip the questioning:
`/dgs:new-project --auto @prd.md` (from a doc) or `/dgs:new-project --auto <spec-id>`
(from a finalized spec).

---

## 6. Define Your First Milestone

```
/dgs:new-milestone
```

Takes over from `new-project`: researches the domain, extracts requirements,
and creates a roadmap of phases. You approve the roadmap, and you're ready to build.

There are two ways to drive this step, both valid:

- **Conversation-driven** (interactive, faster): bare `/dgs:new-milestone` researches
  the domain, extracts requirements, and builds a roadmap of phases through
  conversation with you.
- **Spec-driven** (rigorous, no questioning): `/dgs:add-idea` → `/dgs:write-spec` →
  `/dgs:approve-spec` finalizes a spec first, then `/dgs:new-milestone --auto <spec-id>`
  derives the milestone from it without any interactive questioning.

See [USER-GUIDE.md](USER-GUIDE.md) and [COMMAND-REFERENCE.md](COMMAND-REFERENCE.md) for
the full ideas → spec pipeline.

If a line of work is bigger than a single idea — it'll spawn multiple ideas/todos and
accumulate decisions over time — capture it as a **thread** instead with
`/dgs:add-thread`. See [USER-GUIDE.md § Threads](USER-GUIDE.md#threads).

---

## 7. Build Your First Phase

Four commands, run in a loop for each phase:

1. **`/dgs:discuss-phase 1`** — Capture how you want this phase built before anything
   gets researched or planned. Skip it for reasonable defaults; use it for *your* vision.
2. **`/dgs:plan-phase 1`** — Researches, plans, and verifies atomic task plans for the phase.
3. **`/dgs:execute-phase 1`** — Executes the plans in dependency-ordered waves, one
   atomic commit per task.
4. **`/dgs:verify-work 1`** — Walks you through the deliverables so you can confirm
   the phase does what you expected, with automatic diagnosis if something's off.

Repeat `discuss → plan → execute → verify` for each phase on the roadmap.

---

## 8. Ship

Before `/dgs:complete-milestone`, run the pre-ship gates in order:

1. **`/dgs:code-review branch`** — multi-pass code review across every commit on the
   branch (auto-fixes in local mode).
2. **`/dgs:security-review`** — reads the settled diff as new attack surface (moved
   trust boundaries, newly-reachable sinks, secrets going somewhere new). Reports and
   routes; never auto-fixes. Worth running when the milestone touched auth, network,
   deserialization, or secrets — not routine on every milestone.
3. **`/dgs:audit-milestone`** — verifies the milestone met its definition of done.
4. **`/dgs:plan-milestone-gaps`** — only if the audit found gaps; creates phases to
   close them (loop back to step 7 to build them, then re-run the gates).
5. **`/dgs:adversarial-review milestone`** — the final trust gate; refuter agents
   execute code to refute done-ness claims (no auto-fix).

Then ship:

```
/dgs:complete-milestone
```

Archives the milestone and tags the release. Then run `/dgs:new-milestone` again
to start the next cycle: define → build → ship.

---

## Where to Go Next

- **[SETUP-GUIDE.md](SETUP-GUIDE.md)** — multi-repo setup and internals
- **[USER-GUIDE.md](USER-GUIDE.md)** — full workflow reference
- **[COMMAND-REFERENCE.md](COMMAND-REFERENCE.md)** — every command
