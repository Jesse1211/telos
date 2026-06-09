# Telos

> **0→1 task delivery skill for Claude Code.** A thin orchestrator that wraps `prp-core` and `grill-with-docs` into a single, gated pipeline — from fuzzy idea to delivered code.

## What this skill does

Telos turns "I want to build X" into a delivered branch — but it doesn't reinvent PRDs, plans, or build loops. It **wraps** the tools that already do those well and adds:

- **Triage at the start** — figure out scope/clarity/depth before paying for full PRD work.
- **Hard gates at every design step** — nothing autonomous until you've approved the WHY, the HOW, and the E2E contract.
- **An E2E design document** as the contract — Telos owns this single artifact (no upstream equivalent); the build phase must satisfy it.
- **Project context absorption (once per repo)** — first time you run Telos in a project, it grills the codebase into a `CONTEXT.md` glossary so every subsequent task speaks the project's language.
- **Nestability** — each `/telos task` is independent. Run multiple in the same repo, in sequence or in parallel branches.

What Telos is *not*:

- ❌ A project-management tool (no OKRs, no roadmaps, no post-launch tracking)
- ❌ A PRD/plan/build reimplementation (those belong to `prp-core`)
- ❌ A documentation generator (that belongs to `grill-with-docs`)

## Lifecycle at a glance

```
ABSORB                → grill-with-docs       →  CONTEXT.md, docs/adr/* at repo root
                                                  (skipped on subsequent runs)

P1 PRD                → prp-core:prp-prd      →  .claude/PRPs/prds/<slug>.prd.md
P2 PLAN               → prp-core:prp-plan     →  .claude/PRPs/plans/<slug>.plan.md (decisions inlined; ADR candidates surfaced to P3)
P3 GRILL              → grill-with-docs       →  refines plan, may update CONTEXT/ADRs
P4 E2E DESIGN         → Telos (no delegate)   →  .claude/PRPs/plans/<slug>-e2e.md  ← the contract
P5 BUILD              → prp-core:prp-ralph    →  code + tests/e2e/<slug>.spec.ts + report
P6 DELIVER            → Telos (no delegate)   →  .claude/PRPs/reports/<slug>-final.md
```

Human gates: P1.0 (soft, cheap exit), P1, P2, P3, P4. P5/P6 are autonomous. P4's E2E design doc is frozen at GATE 4 — P5 must satisfy it, not edit it. Before P5 begins, Telos prompts the user to set a `/goal` Stop hook so Claude Code keeps running until the E2E suite passes.

See [`SKILL.md`](./SKILL.md) for the full spec, phase-by-phase.

## Invocation

```
/telos task <description>     # run one full 0→1 lifecycle
/telos task <slug>            # resume an in-flight or completed task by slug
/telos status                 # snapshot of active and recent tasks in this repo
```

A small triage sub-gate (P1.0) lets you abort cheaply if the task isn't what you thought.

## Dependencies

Telos calls these via the **Skill tool**. They must be installed in the Claude environment for the pipeline to work end-to-end. If any is missing, Telos surfaces a blocker rather than silently reimplementing the missing piece.

### `prp-core` (slash commands / agents)

[PRPs Agentic Engineering](https://github.com/Wirasm/PRPs-agentic-eng) — provides the heavy lifting for PRD, plan, and build. Telos targets the command set bundled with the [`claude_md_files`](https://github.com/Wirasm/PRPs-agentic-eng/tree/development/claude_md_files) directory of the `development` branch.

Two relationships:

**Delegated** (Telos invokes via the Skill tool):

| Telos phase | prp-core command | Role |
|---|---|---|
| P1 PRD | `prp-core:prp-prd` | Interactive PRD generation (problem → evidence → hypothesis → MoSCoW → acceptance themes) |
| P2 PLAN | `prp-core:prp-plan` | Codebase intelligence + external research + architect synthesis + task breakdown |
| P5 BUILD | `prp-core:prp-ralph` (preferred) or `prp-core:prp-implement` (fallback) | Autonomous validation loop with bounded retries |

**Handed-off** (Telos surfaces these in P6; user runs them manually — Telos never invokes them):

| Command | Role |
|---|---|
| `prp-core:prp-commit` | Stages files and writes the commit message |
| `prp-core:prp-pr` | Opens the GitHub PR with description |

Also relied on internally by prp-core commands above (Telos doesn't invoke these directly, but they need to be available):

- `prp-core:codebase-explorer` — pattern locator
- `prp-core:codebase-analyst` — data-flow tracer
- `prp-core:web-researcher` — external documentation fetcher

Install location (default): `~/.claude/commands/prp-core/`

### `grill-with-docs` (skill)

[`grill-with-docs`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md) from Matt Pocock's [`skills`](https://github.com/mattpocock/skills) repo — the domain-language grilling session.

Used by:

| Telos phase | grill-with-docs role |
|---|---|
| ABSORB | First-run-in-repo: grill the existing project, write `CONTEXT.md` (glossary) and `docs/adr/*.md` |
| P3 GRILL | Mid-flow: grill the plan against project language and existing ADRs, refine in place |

Telos defers to grill-with-docs's file-shape conventions:

- `CONTEXT.md` at repo root (or `CONTEXT-MAP.md` for multi-context repos) — **glossary only**
- `docs/adr/NNNN-slug.md` — 4-digit sequential, lowercase slug, no `ADR-` prefix

Upstream references:

- [`SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md) — when to grill and how to write CONTEXT.md inline
- [`CONTEXT-FORMAT.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/CONTEXT-FORMAT.md) — glossary file shape
- [`ADR-FORMAT.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/ADR-FORMAT.md) — ADR file shape + criteria

Install location (default): `~/.claude/skills/grill-with-docs/`

### Optional

- `everything-claude-code:e2e` / `everything-claude-code:e2e-runner` — invoked by `prp-ralph` during P5 for E2E scaffolding and flake handling. If unavailable, ralph falls back to writing tests directly per the P4 design doc.

## Tracking & visibility

Telos uses three tiers — no project-level `ROADMAP.md` file, no plan checkboxes.

| Tier | Where | Granularity |
|---|---|---|
| 1. Task-phase | `.claude/PRPs/state/<slug>.json` | which P{n} a slug is at |
| 2. Plan-todo | `.claude/PRPs/reports/<slug>-report.md` (written by `prp-ralph`) | which step inside the plan is running, with status per task |
| 3. Repo-roll-up | `/telos status` output (computed on demand) | all active + recently delivered tasks; surfaces task `done/total` for slugs in P5 |

The plan file (`<slug>.plan.md`) is frozen design intent — it does **not** get checkboxes. The report is the execution trace. `/telos status` aggregates and is the dynamic roadmap.

See [`SKILL.md`](./SKILL.md) → **Tracking Model** for the full algorithm.

## Files Telos produces

```
<repo-root>/
├── CONTEXT.md                                  ← grill-with-docs writes (glossary)
├── CONTEXT-MAP.md                              ← grill-with-docs writes (multi-context only)
├── docs/adr/NNNN-slug.md                       ← grill-with-docs writes (sparingly)
├── tests/e2e/<slug>.spec.ts                    ← prp-ralph writes (P5, per the P4 doc)
│
└── .claude/PRPs/
    ├── prds/<slug>.prd.md                      ← prp-prd writes (P1)
    ├── plans/<slug>.plan.md                    ← prp-plan writes (P2)
    ├── plans/<slug>-e2e.md                     ← Telos writes (P4, the contract)
    ├── plans/completed/                        ← prp-ralph archives plans
    ├── reports/<slug>-report.md                ← prp-ralph writes (P5)
    ├── reports/<slug>-final.md                 ← Telos writes (P6, hand-off summary)
    ├── ralph-archives/                         ← prp-ralph writes (loop state per run)
    └── state/<slug>.json                       ← Telos writes (orchestration + resume)
```

No `.claude/PROJECT/` directory — CONTEXT and ADRs follow `grill-with-docs` convention (repo root + `docs/adr/`).

## Installation

Skills directory layout:

```
~/.claude/skills/telos/
└── SKILL.md
```

Or vendor this directory wholesale next to the dependencies.

Verify `prp-core` and `grill-with-docs` are installed:

```bash
ls ~/.claude/commands/prp-core/
ls ~/.claude/skills/grill-with-docs/
```

## License

MIT — see [`LICENSE`](./LICENSE).
