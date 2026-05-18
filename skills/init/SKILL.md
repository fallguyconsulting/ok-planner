---
name: init
description: "Ensure .ok-planner/ artifact directories exist. Invoked by other ok-planner skills before producing artifacts; also user-invokable as `/init` (idempotent layout check) or `/init --refresh` (update the embedded .ok-planner/CLAUDE.md to the current template, with diff + user confirmation)."
---

# Initialize ok-planner artifact layout

Ensures the `.ok-planner/` directory tree exists at the project root so the
brainstorm, sketch, write-plan, and execute-* skills have somewhere to write
their artifacts. Idempotent: re-running is a no-op.

## Why this skill exists

ok-planner artifacts (specs, plans, sketches, divergence reports) are
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
- `execute-plan` — before dispatching the implementer, and again
  before archiving the plan on completion

Also safe for the user to invoke directly:
- `/init` — ensure the layout exists. Idempotent.
- `/init --refresh` — refresh the embedded `.ok-planner/CLAUDE.md`
  template to match the current skill set. Use when the ok-planner
  skills have been updated and the CLAUDE.md in the project
  predates the new framing. See "Refreshing CLAUDE.md" below.

## Layout

```
.ok-planner/
  CLAUDE.md          # tells agents how to treat this folder; written by this skill
  specs/             # active specs (from brainstorm) — workflow scratch
  plans/             # active plans + their -divergences.md (from write-plan, execute-*) — workflow scratch
  sketches/          # design sketches (from sketch) — workflow scratch
  design/            # durable design docs; bootstrapped by /discover-design,
                     # mutated only by /execute-plan via spec-directed plan tasks
                     # substructure created by /discover-design:
                     #   _discover/        as-is scaffolding
                     #   concepts/         load-bearing nouns (definition, purpose, boundaries, invariants)
                     #   tensions/         catalog of muddy / unspecified / conflicting bits
                     #   review-notes.md   agent-confessed uncertainty for /refine-design
  history/
    specs/           # archived specs (after plan execution completes) — workflow scratch
    plans/           # archived plans + divergence reports (after plan execution completes) — workflow scratch
```

Most subdirectories are workflow scratch — point-in-time records of how
something was conceived. The `design/` subdirectory is the exception:
it's the durable design docs — a concept catalog plus tensions, a
source of truth with the same weight as code. Mutated only through
plan execution. The embedded `CLAUDE.md` explains this distinction
so future agents treat the directories correctly.

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

3. Handle `.ok-planner/CLAUDE.md`:

   **Default (no `--refresh` argument).** If the file does not exist,
   write it with the content below. If it already exists, leave it
   alone — the user may have customized it. Then check the existing
   file for stale-framing markers (see "Refreshing CLAUDE.md"); if
   any are found, surface a one-line notification suggesting
   `/init --refresh`. The notification does not block the rest of
   init from finishing.

   **With `--refresh` argument.** Regenerate the CLAUDE.md from the
   template, show the user a diff against the existing file, and
   only overwrite with the user's explicit confirmation. See
   "Refreshing CLAUDE.md" below for the full flow.

   The point of this file is to give any agent that wanders into
   `.ok-planner/` an explicit, in-folder instruction to leave it
   alone.

   ````markdown
   # .ok-planner — mostly workflow folder, with one durable subdirectory

   This directory holds two kinds of content with different lifecycles
   and different rules for how agents should treat them.

   **Workflow scratch** (`specs/`, `plans/`, `sketches/`, `history/`):
   point-in-time records of how a piece of work was conceived and
   executed. Not living documentation of the codebase. Drift between
   these files and the current code is expected.

   **Durable design docs** (`design/`): the project's canonical
   noun catalog — load-bearing concepts with definitions, purposes,
   boundaries, and invariants — plus a tensions catalog tracking
   what is unresolved. Code references the design (via `@concept:`
   or equivalent annotations at points of enforcement), not the
   other way around. The design docs are **a source of truth with
   the same weight as code**: they describe the project as it
   stands. Like code, they change only through plan execution —
   `execute-plan` is the one skill that mutates them. NOT scratch.

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

   #### When it is OK to touch the workflow scratch

   - The user explicitly asks (e.g. "update the spec at
     .ok-planner/specs/foo.md", "what did we decide about X — check
     the old plan").
   - An ok-planner skill is running and directs you to read or write
     specific files here as part of its documented process.

   In those cases, do exactly what the user or skill asked, then stop.
   Do not expand the scope to "while I'm in here, I'll also fix..."

   ### Durable design docs (`design/`)

   The `design/` subdirectory is **the exception**. It holds the
   project's canonical concept catalog — load-bearing nouns with
   definitions, purposes, boundaries, and invariants — plus a
   tensions catalog tracking what's still muddy.

   **Design docs change only through plan execution, like code.**
   The reason: if the design docs and the code can change
   independently, they drift, and the project ends up in an
   inconsistent state. To prevent that, *all* design-doc changes
   ride the same pipeline as code — a spec captures the change,
   `write-plan` turns it into tasks, and `execute-plan` applies it.

   Skills relate to the design docs in three modes:

   1. **Modifier**: `execute-plan` is the one skill that mutates
      the design docs, carrying out plan tasks that the spec
      directs. (`discover-design` is the bootstrap exception: it
      creates initial design docs for a project that has none, or
      extends the catalog when new areas are discovered. It does
      not modify existing user-edited content.)

   2. **Read-only oracle**: `review-holistic` reads the design
      docs as sacrosanct and uses them to drive whole-codebase
      convergence. There is no per-scope spec for that kind of
      review, so the design docs themselves are the source of
      truth.

   3. **Read-only consumers**: every other skill reads the design
      docs without writing.
      - `brainstorm` and `refine-design` read them to understand
        "how the project is now," and capture any needed changes
        **in the spec they produce** (under a `## Design changes`
        section). The changes get applied later, by
        `execute-plan`.
      - `merge` reads the docs against a fresh merge or rebase
        and **surfaces findings** for the user — mechanical
        issues, drift, semantic conflicts. Resolution is
        human-driven and goes through the spec pipeline; merge
        does not write fixes to the design docs itself.
      - `write-plan` reads to plan spec-directed mutations as
        first-class tasks.
      - `review-work` and `review-plan` read to verify that
        spec-directed mutations landed correctly.

   Layout (created by `/discover-design`):
   - `_discover/` — as-is scaffolding. Wide, detailed, possibly
     redundant. Discarded once the design stabilizes.
   - `concepts/` — one concept per file. Mutated in place by
     `execute-plan` per the spec's `## Design changes` section;
     Notes section is append-only.
   - `tensions/` — one tension per file. Tensions move to
     `tensions/_resolved/` when `execute-plan` carries out a
     resolution-bearing spec.
   - `review-notes.md` — agent-confessed uncertainty from the
     `/discover-design` run (judgment calls, suspected concepts,
     unresolved fix-loop issues). Consumed by `/refine-design`;
     not durable.

   - **`review-holistic` consults `design/concepts/`** when forming
     judgments — a finding that contradicts a documented boundary
     or invariant needs an explicit "the documented concept is
     wrong because..." rather than a flag against the code.
     `brainstorm` and `refine-design` consult to understand current
     state; the spec captures any needed changes. Other skills do
     not consult as oracle.
   - **Concept files (`concepts/<slug>.md`) are MUTABLE during
     `execute-plan`.** Editing Definition / Purpose / Boundaries /
     Invariants in place is the normal mode — this is the
     prescriptive design, not an ADR journal. The Notes section is
     the append-only audit trail. Mutations ride alongside the
     code changes that conform to them and stay reversible until
     the plan executes.
   - **DO NOT delete concept files on your own initiative**, even
     if they look stale. Use `/merge` to surface drift; resolution
     goes through the spec pipeline.
   - **DO NOT mutate `tensions/` entries outside the spec
     pipeline** — those entries record what the project considers
     unresolved.

   #### Consulting the design docs: read `concepts.md` first

   When `review-holistic`, `brainstorm`, or `refine-design` is
   reading the design docs, the lookup pattern is:

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

   #### `@concept:` annotations

   `@concept: <slug>` annotations in source code are inline
   citations marking sites where a concept is enforced. They
   make the next agent's lookup cheap. Same granularity
   discipline as `@blessed-invariant`: annotate where the concept
   is enforced or expressed, not every file that happens to touch
   it. No carpet-bombing.

   - **`execute-plan`** adds or modifies annotations when a plan
     task explicitly directs (the spec usually surfaces this when
     a concept is being introduced or changed).
   - **`discover-design`** can seed initial annotations on
     bootstrap.
   - **`review-holistic`** notes "consulted concept: X
     (annotation missing at <site>)" in its findings; a
     subsequent `execute-plan` run leaves the annotation when
     applying the fix.

   No other skill annotates on its own. Annotations accrete
   through `execute-plan` runs.

   ## Layout

   - `specs/` — active specs from `/brainstorm` and
     `/refine-design` (workflow scratch)
   - `plans/` — active plans from `/write-plan`, plus their
     `-divergences.md` reports written by `/execute-plan`'s
     divergence auditor (workflow scratch)
   - `sketches/` — design sketches from `/sketch` (workflow scratch)
   - `design/` — durable design docs (concepts + tensions; mutated
     only by `/execute-plan` via spec-directed plan tasks;
     bootstrapped by `/discover-design`)
   - `history/specs/` and `history/plans/` — specs and plans
     archived here automatically when an execute-* skill finishes
     a plan (workflow scratch)
   ````

4. Report back to the calling skill (or user, if invoked directly) with
   one short line stating what was created vs. already present. Do not
   produce a long explanation.

## Refreshing CLAUDE.md

The embedded `.ok-planner/CLAUDE.md` template evolves alongside the
ok-planner skills. When the skill set is updated (terminology
changes, new conventions, etc.), an existing project's CLAUDE.md
can fall behind and contradict the current skills. This section
covers detecting that gap and bringing the file up to date.

### Stale-framing markers

The current template uses specific terminology that earlier
templates did not. If an existing `.ok-planner/CLAUDE.md` contains
any of these literal strings, the file predates the current
framing:

- `design log` (the durable docs are now "design docs")
- `implementation notes` or `-notes.md` (the post-execution
  artifact is now the divergence report, `-divergences.md`)
- `Tensions resolved` (the spec section is now `## Design changes`,
  unifying tension and concept mutations)
- "any agent ... should consult" framing for the design docs
  (consultation is now scoped to specific skills, not blanket)

When a default `/init` run sees one of these markers in the
existing CLAUDE.md, surface a one-line notification:

  > Your `.ok-planner/CLAUDE.md` predates the current skill
  > framing — run `/init --refresh` to update it (you'll review
  > a diff before any overwrite).

Do not block. Do not auto-update. The user decides when to refresh.

### `/init --refresh` flow

When invoked with `--refresh`:

1. Read the existing `.ok-planner/CLAUDE.md` from disk. If the
   file does not exist, treat this as a default `/init` (write
   the template fresh, report, done).
2. Generate the current template content as a string (the same
   content shown above in step 3 of the Process section).
3. Show the user a unified diff between the existing file and the
   new template (use `diff -u` via Bash on temp files, or render
   the diff inline if the file is small). Make sure the diff is
   actually visible — this is the user's chance to spot
   customizations that would be lost.
4. Ask the user: "Overwrite, keep current, or merge manually?"
   - **Overwrite**: write the new template to
     `.ok-planner/CLAUDE.md`, replacing the existing content
     entirely.
   - **Keep current**: do nothing. The file stays as-is.
   - **Merge manually**: print the new template content in a
     fenced code block, tell the user to copy-paste the parts they
     want, and exit without writing. The user handles the merge in
     their own editor.
5. Report the outcome in one line ("CLAUDE.md refreshed."
   / "CLAUDE.md unchanged." / "New template printed; merge
   manually.").

Custom edits to a CLAUDE.md are not migrated automatically. The
diff exists so the user can see what they'd lose; if there are
custom sections worth keeping, the "merge manually" path is the
right one. This is intentionally low-tech — refreshes should be
rare and worth a few minutes of attention from the user.

## Reporting

Keep output minimal. Examples:

- First run: "Initialized `.ok-planner/` layout."
- Subsequent runs: "`.ok-planner/` layout already present."
- Partial: "Created `.ok-planner/sketches/` and `.ok-planner/history/`;
  rest already present."
- Stale CLAUDE.md detected: append "Note: `.ok-planner/CLAUDE.md`
  predates the current framing — run `/init --refresh` to update."
- After `/init --refresh` overwrite: "CLAUDE.md refreshed."
- After `/init --refresh` keep-current: "CLAUDE.md unchanged."
- After `/init --refresh` merge-manually: "New template printed;
  merge manually."

The calling skill should continue immediately after this skill reports.

## What this skill does NOT do

- Does not modify `.gitignore`. Whether `.ok-planner/` is tracked in git
  is the user's decision.
- Does not delete or move existing files.
- Does not validate the contents of existing artifacts.
- Does not rename or migrate artifacts from other locations
  (e.g., `docs/specs/` from older versions of these skills). Migration is
  a one-time user-driven task, not something this skill handles silently.
