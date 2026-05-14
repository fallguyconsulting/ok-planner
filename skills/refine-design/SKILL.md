---
name: refine-design
description: "ONLY activated by explicit /refine-design slash command. Never auto-triggered by conversation content."
---

# Refine Design

Specialization of `brainstorm` for resolving design tensions. The user
picks tensions to address, picks a resolution shape per tension, and
the skill hands off to `brainstorm` to produce a spec that covers both
the concept-doc mutations and the code reconciliation. From the spec
forward, the standard `brainstorm` → `write-plan` → `execute-plan`
pipeline applies.

<HARD-GATE>
Do NOT mutate `concepts/`, move tension files, or write code during
the intake. Concept-file mutations and code changes happen during
`execute-plan`, after the spec and plan are approved. The intake's
job is to set up the brief for `brainstorm`, nothing more.
</HARD-GATE>

<DIALOGUE-REQUIRED>
The intake (tension selection, resolution-shape picking, review-notes
walk) is a dialogue with the user. The user is the design oracle —
they make the calls. Do not pick resolution shapes on their behalf,
do not "interpret" a hesitant answer as a yes, do not skip review-notes
entries on the user's behalf. Auto-mode does not authorize design
decisions; it only silences mechanical permission prompts.
</DIALOGUE-REQUIRED>

## Why this exists

The design log under `.ok-planner/design/` is the project's canonical
noun catalog. Code references the design at points of enforcement
(`@concept: <slug>` annotations, or equivalent); the design owns the
definition. When a concept's boundary drifts from the code, that's a
defect to fix — but the fix is rarely *just* the code or *just* the
concept doc. Usually it's both: the concept gets sharpened, the code
gets brought into line, the alias gets retired across both surfaces.

`brainstorm` is the project's tool for "I have a change in mind, help
me design it." `refine-design` is brainstorm pointed at the tensions
catalog: the change in mind is "resolve these tensions," and the
resulting spec covers both the design-doc work and the code work as
one coherent piece.

This isn't a separate pipeline. It's brainstorm with a specialized
intake.

## Relationship to brainstorm

`refine-design` owns:
- Loading the tensions catalog + relevant concept files + review-notes
- Walking `review-notes.md` with the user
- Letting the user pick which tensions to address this session
- Letting the user pick a resolution shape per selected tension
- Marking the selected tensions as `status: resolving` with a pointer
  to the (about-to-be-written) spec
- Constructing a brief for `brainstorm` that captures what's been
  decided

`brainstorm` owns everything from there:
- Clarifying questions
- Design proposal
- Spec writing (with the additional "Tensions resolved" section)
- Spec review loop
- User approval
- Transition to `write-plan`

`refine-design` does NOT duplicate brainstorm's machinery. It sets up
context and invokes `ok-planner:brainstorm`.

## Inputs

- `.ok-planner/design/tensions/` — open tensions to choose from.
- `.ok-planner/design/concepts/` — read for context; mutations happen
  during `execute-plan`, not now.
- `.ok-planner/design/review-notes.md` — agent-confessed uncertainty
  from the last `discover-design` run.
- `.ok-planner/design/_discover/` — referenced for evidence during the
  walk.

## When to invoke

- After `discover-design` has produced `concepts/` and `tensions/`,
  and the user is ready to start resolving.
- When new tensions are added (by a later `discover-design` re-run or
  by a `brainstorm` spec that surfaced an unresolved concept question)
  and the user wants to address some.

## Process

1. Run `ok-planner:init` so the layout exists.
2. Verify `.ok-planner/design/concepts/` and
   `.ok-planner/design/tensions/` exist. If either is empty, tell the
   user to run `/discover-design` first and stop.
3. **Walk `review-notes.md` if present.** Surface entries section by
   section. Per entry the user can:
   - Confirm the extractor's call (no action).
   - Add the item to this session's tension queue (which tension(s)
     it implies — promote a new tension, or attach to an existing one).
   - Defer (leave the review-note as-is for a future session).
   - Reject the observation (annotate review-notes with a strikethrough
     + one-line reason).

   Done when every section has been touched. After the walk, rotate
   `review-notes.md` to `review-notes-<date>.md` so the next
   `discover-design` run starts clean.

4. **Tension selection.** Show the user the open tensions catalog
   grouped by category. Ask which they want to address this session.
   Defaults: smallest-first, by-affected-concept, by-category, or
   freeform pick. The user names tensions; the skill records them as
   the session queue.

5. **Resolution-shape walkthrough.** For each tension in the session
   queue:
   - Display the tension entry (What is muddy / Why it matters /
     Resolution candidates).
   - Quote the relevant concept file(s) and `_discover/` material.
   - Present resolution shapes — the extractor's candidates, or new
     ones if the user wants. Each shape should be stated as a
     concrete change to one or more concept docs AND the code path(s)
     that need reconciling.
   - The user picks. Possible verdicts:
     - **pick shape X** — record the pick.
     - **reject the tension** — move the file to `tensions/_rejected/`
       with `status: rejected` and a one-line `rejection-reason`.
       Drop from the session queue. (This IS a mutation, but it's a
       tension-state mutation, not a concept mutation, and reflects
       a user decision that doesn't need a spec.)
     - **defer this one** — leave the tension as `status: open` and
       drop from the session queue.

6. **Mark resolving + construct brief.** For each tension still in
   the queue after step 5:
   - Update its frontmatter to `status: resolving` with `spec:
     <upcoming-spec-slug>`. The slug is generated from the date plus a
     short summary of the cluster (e.g., `2026-05-10-vocabulary-cleanup`).
   - Build a brief in the conversation that names: each tension being
     resolved, the picked resolution shape, the affected concept
     files, the code paths that need reconciling, the aliases being
     retired (if any).

7. **Hand off to brainstorm.** Invoke `ok-planner:brainstorm`. The
   brief established in step 6 is the seed context — brainstorm
   recognizes that the goal is already established (per its
   "Establish the goal" step) and proceeds directly to clarifying
   questions / design / spec writing.

   The spec brainstorm writes will include an additional section,
   **Tensions resolved**, listing each tension being addressed with
   its resolution shape, target concept-file edits, and code
   reconciliation. The rest of the spec (architecture, components,
   testing strategy) covers the code work as brainstorm normally
   handles it.

8. **brainstorm's downstream flow.** From step 7 onward, the user is
   in brainstorm's flow: spec review subagent, user spec review,
   skip-or-walk the design-log step (skip when the spec has a
   "Tensions resolved" section — those impacts are already specified
   in the spec, not surfaced as candidate updates), transition to
   `write-plan`.

9. **What happens during execute-plan.** When the plan eventually
   executes, it applies the code changes per the plan tasks AND the
   concept-file mutations specified in the spec's "Tensions resolved"
   section. As each tension's work completes, `execute-plan` moves
   the tension file from `tensions/<slug>.md` to
   `tensions/_resolved/<slug>.md` with `status: resolved` and a
   `resolution` block summarizing the outcome.

## Tension lifecycle

```
open                           ← state after discover-design
  ↓ user picks resolution shape in refine-design step 5
resolving (spec: <slug>)       ← awaiting spec approval + plan execution
  ↓ execute-plan applies the resolution
resolved                       ← in tensions/_resolved/

(alternative paths)
open → rejected                ← user rejected during refine-design intake
open → open                    ← user deferred
resolving → open               ← spec/plan abandoned (rare; user reverts)
```

## Session shape

A refine-design session produces **one spec**, just like brainstorm.
If the session queue is broader than one spec can cover well, surface
that to the user — they can either drop tensions from the queue or
defer the broader scope to a follow-up session. Do not unilaterally
split into multiple specs.

A session can resolve a single tension (small fix) or a cluster of
related tensions (the three `Store` / `ClaimProducer` aliases are one
cleanup; the three `frame_resolution` / `mode` vocabulary tensions
are one cleanup; etc.).

## Mutability discipline (for execute-plan to follow)

When `execute-plan` applies the spec:
- Concept files in `concepts/` are mutated in place. Update sections
  freely; the Notes section gets a new entry citing the spec slug
  (`Tension resolved: <tension-slug> — <one-line>`).
- Resolved tensions move to `tensions/_resolved/<slug>.md`.
- Rejected tensions (from refine-design intake) are already in
  `tensions/_rejected/`.
- New tensions surfaced during brainstorm's dialogue are written to
  `tensions/` as `status: open`, as per brainstorm's
  tension-impact step.

## What this skill does NOT do

- Doesn't mutate `concepts/` directly. Mutations happen in
  `execute-plan` after spec + plan approval, so the design changes
  ride alongside the code changes and stay reversible until then.
- Doesn't write code. Brainstorm produces the spec; write-plan
  sequences the work; execute-plan does the writing.
- Doesn't re-do discovery. If the user finds that a tension can't be
  resolved because discovery was thin in some area, they exit and run
  `/discover-design` to back-fill, then return.
- Doesn't auto-pick resolution shapes. Every shape requires a user
  verdict, full stop.
- Doesn't introduce code annotations. If the project adopts a
  `@concept:` convention, that's a separate brainstorm in its own
  right.
