---
name: init
description: "Ensure the project's .ok-planner/ artifact directories exist. Invoked by other ok-planner skills before they produce documents. Not user-facing."
---

# Initialize ok-planner artifact layout

Ensures the `.ok-planner/` directory tree exists at the project root so the
brainstorm, sketch, write-plan, and execute-* skills have somewhere to write
their artifacts. Idempotent: re-running is a no-op.

## Why this skill exists

ok-planner artifacts (specs, plans, sketches, implementation notes) are
**workflow scratch**, not project documentation. They describe how a piece
of work was conceived and executed at a point in time — they are not
intended to be kept in sync with the codebase as it evolves.

Putting them under a dotfile-prefixed directory (`.ok-planner/`) signals
this clearly to future agents: this is the planner's scratchpad, not docs
the agent should helpfully update during unrelated work.

## When invoked

Called by other ok-planner skills as their first step when they're about to
produce or move artifacts:

- `brainstorm` — before writing a spec
- `sketch` — before writing a sketch
- `write-plan` — before writing a plan
- `execute-plan` — before creating the implementation notes
  file, and again before archiving on completion

Also safe for the user to invoke directly via `/init` if they want the
layout created without starting a workflow.

## Layout

```
.ok-planner/
  CLAUDE.md          # tells agents how to treat this folder; written by this skill
  specs/             # active specs (from brainstorm) — workflow scratch
  plans/             # active plans + their -notes.md (from write-plan, execute-*) — workflow scratch
  sketches/          # design sketches (from sketch) — workflow scratch
  design/            # durable design log; managed by /discover-design + /refine-design + /merge
                     # substructure created by /discover-design:
                     #   _discover/        as-is scaffolding
                     #   concepts/         load-bearing nouns (definition, purpose, boundaries, invariants)
                     #   tensions/         catalog of muddy / unspecified / conflicting bits
                     #   review-notes.md   agent-confessed uncertainty for /refine-design
  history/
    specs/           # archived specs (after plan execution completes) — workflow scratch
    plans/           # archived plans + notes (after plan execution completes) — workflow scratch
```

Most subdirectories are workflow scratch — point-in-time records of how
something was conceived. The `design/` subdirectory is the exception:
it's the durable design log — a concept catalog plus tensions, intended
to be consulted during review and to outlive any one plan. The embedded
`CLAUDE.md` explains this distinction so future agents treat the
directories correctly.

## Process

1. Resolve the project root. Use the working directory the user invoked
   Claude Code from (`pwd`). If a `.git` directory is found at or above
   that, use the directory containing `.git` as the root. Otherwise use
   the working directory.

2. Create any missing directories:
   - `.ok-planner/specs/`
   - `.ok-planner/plans/`
   - `.ok-planner/sketches/`
   - `.ok-planner/design/`
   - `.ok-planner/history/specs/`
   - `.ok-planner/history/plans/`

   Use `mkdir -p` so existing directories are left untouched.

3. If `.ok-planner/CLAUDE.md` does not exist, write it with the content
   below. If it already exists, leave it alone — the user may have
   customized it. The point of this file is to give any agent that
   wanders into `.ok-planner/` an explicit, in-folder instruction to
   leave it alone.

   ````markdown
   # .ok-planner — mostly workflow folder, with one durable subdirectory

   This directory holds two kinds of content with different lifecycles
   and different rules for how agents should treat them.

   **Workflow scratch** (`specs/`, `plans/`, `sketches/`, `history/`):
   point-in-time records of how a piece of work was conceived and
   executed. Not living documentation of the codebase. Drift between
   these files and the current code is expected.

   **Durable design log** (`design/`): the project's canonical noun
   catalog — load-bearing concepts with definitions, purposes,
   boundaries, and invariants — plus a tensions catalog tracking what
   is unresolved. Code references the design (via `@concept:` or
   equivalent annotations at points of enforcement), not the other
   way around. Intended to be consulted during code review and to
   outlive any single plan. NOT scratch.

   ## Default behavior for agents

   ### Workflow scratch (`specs/`, `plans/`, `sketches/`, `history/`)

   Unless the user or an active skill (e.g. `/brainstorm`,
   `/write-plan`, `/execute-plan`, `/review-plan`, `/sketch`)
   explicitly directs you here, ignore these subdirectories:

   - **Do not consult these files to understand the project.** They
     reflect what someone was thinking at a moment in time. The
     codebase is the source of truth; these artifacts are not.
   - **Do not include `.ok-planner/` files in general repository
     exploration**, codebase walkthroughs, or "how does this project
     work" research. Skip them the same way you would skip a build
     directory.
   - **Do not propose updating, refreshing, or reconciling these files
     with the current state of the code.** Drift between an old plan
     and the current code is expected and fine. The artifact stays as
     it was written.
   - **Do not edit, rename, move, or delete files here on your own
     initiative**, even if they look stale, redundant, or wrong.

   ## When it is OK to touch the workflow scratch

   - The user explicitly asks (e.g. "update the spec at
     .ok-planner/specs/foo.md", "what did we decide about X — check
     the old plan").
   - An ok-planner skill is running and directs you to read or write
     specific files here as part of its documented process.

   In those cases, do exactly what the user or skill asked, then stop.
   Do not expand the scope to "while I'm in here, I'll also fix..."

   ### Durable design log (`design/`)

   The `design/` subdirectory is **the exception**. It holds the
   project's canonical concept catalog — load-bearing nouns with
   definitions, purposes, boundaries, and invariants — plus a
   tensions catalog tracking what's still muddy. These are intended
   to be consulted — actively — during code review, when forming
   review findings, when weighing whether something is a defect or a
   stated boundary of a concept.

   Layout (created by `/discover-design`):
   - `_discover/` — as-is scaffolding. Wide, detailed, possibly
     redundant. Discarded once the design stabilizes.
   - `concepts/` — one concept per file. Mutable in body; Notes
     section is append-only.
   - `tensions/` — one tension per file. Resolved by
     `/refine-design`; resolved tensions move to
     `tensions/_resolved/`.
   - `review-notes.md` — agent-confessed uncertainty from the
     `/discover-design` run (judgment calls, suspected concepts,
     unresolved fix-loop issues). Consumed by `/refine-design`;
     not durable.

   - **DO consult `design/concepts/` when reviewing code.** A finding
     that contradicts a documented boundary or invariant needs an
     explicit "the documented concept is wrong because..." rather
     than a flag against the code.
   - **Concept files (`concepts/<slug>.md`) are MUTABLE.** Editing
     Definition / Purpose / Boundaries / Invariants in place is the
     normal mode — this is the prescriptive design, not an ADR
     journal. The Notes section is the append-only audit trail; each
     tension resolution adds a Notes entry. Mutations happen during
     `execute-plan` of a refine-design-originated spec, ride alongside
     the code changes that conform to them, and stay reversible until
     the plan executes.
   - **DO NOT delete concept files on your own initiative**, even if
     they look stale. Use `/merge` to surface drift; the user
     decides whether to retire a concept or fold it into another.
   - **DO NOT mutate `tensions/` entries except via `/refine-design`**
     or by explicit user request — those entries record what the
     project considers unresolved.

   ## Consulting the design log: read `concepts.md` first

   When the design log exists, any agent working on the project's
   code (regardless of which skill is active) should:

   1. **Read `.ok-planner/design/concepts.md` first** — a small
      auto-generated TOC of every concept with a one-sentence
      definition. It is the single artifact you can read in one
      shot to know what concepts the project traffics in.
   2. **Grep for `@concept:` annotations in the code under
      consideration** — these are inline citations that say "this
      site implements concept X." Use `rg '@concept:' <path>` to
      find them locally; use `rg '@concept: <slug>'` to find every
      enforcement site for a specific concept.
   3. **Read `concepts/<slug>.md`** for the full definition
      (purpose, boundaries, invariants) for any concept surfaced
      by step 1 or 2 that's load-bearing for the work at hand.

   ## Annotate-on-consult: leave the trail

   If you consulted a concept to understand or modify a piece of
   code, **leave a `@concept: <slug>` annotation** at the
   most-specific load-bearing site before the next agent reads the
   file cold. Same granularity discipline as `@blessed-invariant`:
   annotation marks where the concept is enforced or expressed, not
   every file that happens to touch it. No carpet-bombing.

   - **Write-permitted agents** (editing source files in this
     session) leave the annotation directly.
   - **Read-only agents** (review-* skills, exploration) note
     "consulted concept: X (annotation missing at <site>)" in
     their findings or output so a subsequent fixer can leave the
     annotation as part of the fix.

   This rule replaces any need for a separate "annotate the
   codebase" task — annotations accrete through normal work.
   Untouched code stays bare; grep-discoverability still works for
   whatever subset has been annotated.

   The `design/` log is managed by `/discover-design` (bootstrap and
   expand the as-is + tensions catalog), `/refine-design` (specialized
   brainstorm that produces a spec covering both concept-doc
   mutations and code reconciliation for a chosen set of tensions),
   and `/merge` (post-merge reconciliation). `/brainstorm` may add
   new concept updates or tension entries when a spec surfaces them.

   ## Layout

   - `specs/` — active specs from `/brainstorm` (workflow scratch)
   - `plans/` — active plans from `/write-plan`, plus their
     `-notes.md` implementation notes written during execution
     (workflow scratch)
   - `sketches/` — design sketches from `/sketch` (workflow scratch)
   - `design/` — durable design log (concepts + tensions; managed by
     `/discover-design`, `/refine-design`, `/merge`)
   - `history/specs/` and `history/plans/` — specs and plans archived
     here automatically when an execute-* skill finishes a plan
     (workflow scratch)
   ````

4. Report back to the calling skill (or user, if invoked directly) with
   one short line stating what was created vs. already present. Do not
   produce a long explanation.

## Reporting

Keep output minimal. Examples:

- First run: "Initialized `.ok-planner/` layout."
- Subsequent runs: "`.ok-planner/` layout already present."
- Partial: "Created `.ok-planner/sketches/` and `.ok-planner/history/`;
  rest already present."

The calling skill should continue immediately after this skill reports.

## What this skill does NOT do

- Does not modify `.gitignore`. Whether `.ok-planner/` is tracked in git
  is the user's decision.
- Does not delete or move existing files.
- Does not validate the contents of existing artifacts.
- Does not rename or migrate artifacts from other locations
  (e.g., `docs/specs/` from older versions of these skills). Migration is
  a one-time user-driven task, not something this skill handles silently.
