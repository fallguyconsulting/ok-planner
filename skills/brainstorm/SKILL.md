---
name: brainstorm
description: "ONLY activated by explicit /brainstorm slash command. Never auto-triggered by conversation content."
---

# Brainstorming Ideas Into Designs

Turn ideas into specs through collaborative dialogue. Understand context, ask questions, present design, get approval.

<HARD-GATE>
Do NOT write code, scaffold, or take implementation action until the user approves a design. The design can be short for simple projects, but it must exist and be approved.
</HARD-GATE>

<DIALOGUE-REQUIRED>
This skill is a dialogue with the user. The clarifying questions, section-by-section approvals, spec review, and handoff to write-plan are not optional. Specifically, do not:

- Collapse the intake into assumptions and proceed
- Bundle multiple questions into one message — one question per message, wait for the answer
- Pick between options you have offered the user on their behalf — present the options, then wait
- Move past a question without a user answer in hand
- Chain into `write-plan` before the user has approved the written spec

The product of this skill is a user-approved spec — that requires the user. General "run unsupervised" guidance does not apply here; this skill *is* the dialogue.
</DIALOGUE-REQUIRED>

## Process

1. **Initialize layout** -- invoke `ok-planner:init` so `.ok-planner/specs/` exists before any writing starts.
2. **Establish the goal** -- if context from the conversation makes it clear what to brainstorm, proceed. Otherwise ask the user what they want to design. If invoked from `ok-planner:refine-design`, the goal has already been established in the conversation (a brief naming: the session's tension queue with picked resolution shapes, any tensions rejected during intake, affected concept files, code paths needing reconciliation, review-notes outcomes, and a proposed spec slug) — recognize that brief as the goal and proceed. Every item in the brief that touches the design docs must end up under `## Design changes` in the spec.
3. **Explore project context** -- check files, docs, recent commits, and (if it exists) the design docs under `.ok-planner/design/` to understand "how the project is now"
4. **Ask clarifying questions** -- one at a time, prefer multiple choice
5. **Propose 2-3 approaches** -- with trade-offs and your recommendation
6. **Present design** -- scaled to complexity, get user approval section by section. Surface any design-doc impacts (concept additions/mutations, tension resolutions or new tensions) as part of the discussion; they go into the spec, not into the design docs directly.
7. **Write spec** -- save to `.ok-planner/specs/YYYY-MM-DD-<topic>-design.md`. If the work touches the design docs, include a `## Design changes` section enumerating the mutations execute-plan will apply.
8. **Spec review** -- dispatch reviewer subagent, fix issues, re-review until clean
9. **User reviews spec** -- ask user to review before proceeding
10. **Transition** -- invoke ok-planner:write-plan

## What a spec is

A brainstorm produces **one spec**, capturing the full scope the user is
designing. Do not split into multiple specs. If the scope is broader than one
brainstorm can cover well, surface that to the user — they decide whether to
narrow the brainstorm or keep going.

A spec describes **what to build**: architecture, components, data flow,
behavior, error handling, testing strategy. It does not describe **how to
ship it**. No PRs, commits, branches, merge strategies, deployment steps,
release plans, or rollout phases. Delivery process is out of scope — the
user owns that.

## Asking Questions

- One question per message
- Prefer multiple choice when possible — presented as plain text (A, B, C…) with a recommendation. Free-form replies are always fine.
- Do **not** use tools that constrain the user's reply to a predetermined structure (e.g. `AskUserQuestion`, poll-style pickers, forced single-select widgets). Ask in prose so the user can answer in prose, pick a letter, push back, reframe the question, or go off-script.
- Focus on purpose, constraints, success criteria
- Scope is the user's call. If it feels unclear or sprawling, ask — do not unilaterally decompose into sub-projects.

## Presenting the Design

- Scale each section to its complexity
- Ask after each section whether it looks right
- Cover: architecture, components, data flow, error handling, testing
- YAGNI ruthlessly

## Design docs awareness (read-only)

If `.ok-planner/design/concepts/` exists, it is the project's
canonical concept catalog and a source of truth equal in weight
to the code. **Brainstorm is read-only against the design docs.**
Mutations happen later, via `execute-plan` carrying out a
spec-directed plan. Read the docs now; capture changes in the
spec; let the pipeline apply them.

During step 3 (Explore project context):

1. Read `.ok-planner/design/concepts.md` — the auto-generated TOC,
   always small, always one-shot-readable. Now you know what
   concepts exist.
2. Grep for `@concept:` annotations in the area you're exploring
   (`rg '@concept:' <path>`). Each marks a load-bearing site.
3. Read `concepts/<slug>.md` in full for any concept surfaced by
   step 1 or 2 that the brainstorm's subject area touches.
4. Check `.ok-planner/design/tensions/` for open tensions in the
   same area — they may already describe the muddiness the
   brainstorm wants to address, in which case the user might want
   to run `/refine-design` for those tensions instead of (or
   alongside) this brainstorm.

Use the catalog's terms in the design and respect its stated
boundaries and invariants. If the design would change a concept's
shape, add a new concept, retire one, or resolve/raise a tension,
capture that as a `## Design changes` entry in the spec (see
"Capturing design changes in the spec" below). Do not mutate
`concepts/`, `tensions/`, or any other file under
`.ok-planner/design/` during brainstorm — those are execute-plan's
to touch.

If `.ok-planner/design/concepts/` doesn't exist, this skill
operates without the design-docs context. Do not create it as a
side effect; do not suggest `/discover-design` unless the user
asks.

## Working in Existing Codebases

- Explore current structure before proposing changes. Follow existing patterns.
- When the spec introduces shape-bearing entities (new persistence tables, new HTTP routes, new wire formats, new event kinds, new error envelopes), check the conventions in place for similar entities and match them in the spec — column naming, route plurality, payload shapes, prefix discipline. Downstream readers assume convention; surprise mismatches end up baked into the plan and the code.
- Where existing code has problems that affect the work, include targeted improvements
- Don't propose unrelated refactoring

## Capturing design changes in the spec

If the work touches the design docs, the spec must say so
explicitly under a `## Design changes` section. Brainstorm does
**not** mutate `.ok-planner/design/` directly — the design docs
are a source of truth with the same weight as code, and like
code they change only through plan execution. Capturing changes
in the spec lets `write-plan` turn them into first-class plan
tasks and `execute-plan` apply them alongside the code.

If `.ok-planner/design/concepts/` does not exist, skip this
section entirely — the project has not bootstrapped design docs
and this brainstorm is not the place to do it. Do not mention
the absence; do not suggest `/discover-design` unless the user
asks.

While presenting the design (step 6), identify two kinds of
impact:

**Concept impacts** — the spec sharpens, splits, merges,
introduces, or retires a concept. Candidate if ANY of these:
- It introduces a new load-bearing noun the project did not have.
- It changes the boundary of an existing concept (what's in vs.
  what's now in a neighbor).
- It retires a concept, or retires an alias that should no longer
  be used.
- It mutates Definition / Purpose / Boundaries / Invariants of
  an existing concept.

**Tension impacts** — the spec touches the tensions catalog.
Candidate if ANY of these:
- It resolves an open tension (record which one and how).
- It surfaces a new tension the project should catalog (a choice
  the spec doesn't resolve, an inconsistency, an
  acknowledged-but-unresolved tradeoff).

A candidate does NOT belong if it's a purely local implementation
detail. The bar: would a reviewer benefit from knowing this when
forming findings?

Walk each candidate with the user — confirm-as-is / edit /
reject. For accepted candidates, write a `## Design changes`
section in the spec with one bullet per change, each precise
enough that execute-plan can apply it mechanically. Example:

```
## Design changes

- Concept: mutate `concepts/store.md` in place. Replace the
  Boundaries section with: ... (full new text). Append a Notes
  entry: `<date> — Boundary narrowed per spec <spec-slug> to
  exclude in-memory mocks.`
- Concept: create `concepts/claim-producer.md` from the template
  in `ok-planner:discover-design`'s SKILL.md, with Definition: ...,
  Purpose: ..., Boundaries: ..., Invariants: ....
- Tension: resolve `tensions/store-vs-claim-producer.md`. Move
  the file to `tensions/_resolved/store-vs-claim-producer.md`
  with `status: resolved` and a `resolution:` block summarizing
  the outcome (the resolution shape is: ...).
- Tension: create `tensions/cache-invalidation.md` from the
  template in `ok-planner:discover-design`'s SKILL.md, with
  status: open and the muddiness described as: ....
```

If the spec produced no design-doc impacts (small specs often
won't), don't add a `## Design changes` section.

## Writing the spec and reviewing it

Write the spec to `.ok-planner/specs/YYYY-MM-DD-<topic>-design.md` (user preferences override this path). Dispatch a spec reviewer subagent:

```
Agent (general-purpose):
  Read the spec at [path].

  ## Ground every claim in source code

  Before checking the spec internally, verify it against the codebase.
  Hallucinated current-state is the most expensive failure
  downstream — a spec that confidently references things that do
  not exist sends an implementer chasing ghosts. Catch it here.

  - Read every file path the spec names. If a path does not exist
    and the spec does not mark it as new, flag it with the path.
  - For every function, class, module, type, table, endpoint, or
    other named entity the spec references as existing, find it
    (`rg`, `grep`, or direct read). If it does not exist, or its
    actual shape differs materially from what the spec assumes,
    flag it with file:line.
  - For every "currently X" / "today the code does Y" /
    "the existing Z" assertion, verify against the actual code.
    Flag any that do not match, citing the file and what is
    actually there.
  - For every NEW shape-bearing entity the spec introduces (new
    persistence table, new HTTP route surface, new wire format,
    new event kind, new error envelope), find its closest cousin
    in the codebase and check that the spec's proposed shape
    matches the prevailing convention — column naming, route
    plurality, payload structure, prefix discipline. The planner
    downstream will codify the proposed shape as written; a spec
    that names a new table inconsistently with its siblings, or
    proposes a route shape that contradicts the rest of the API,
    sends the cluster downstream into the plan and the code. If
    the deviation is intentional and the spec justifies it, fine;
    if the spec is silent, flag the mismatch.

  If you find one such issue, keep looking — they cluster.
  Exhaustively check every file path and every named entity; do
  not sample.

  ## Check against the design docs (if present)

  If `.ok-planner/design/concepts/` exists, consult it. Brainstorm
  is where specs are written against the design docs, so this
  review is the natural place to catch mismatches before the spec
  is approved. Downstream skills do not re-check the spec against
  the design docs.

  1. Read `.ok-planner/design/concepts.md` (the TOC).
  2. For concepts the spec touches by name or by area, read
     `concepts/<slug>.md` in full.
  3. Check `.ok-planner/design/tensions/` for open tensions in the
     same area.

  Flag:
  - The spec uses a load-bearing noun differently than its concept
    file defines it (or aliases over a documented term without
    saying so).
  - The spec proposes work that would violate a stated invariant
    or cross a stated boundary, without addressing the change in
    a `## Design changes` section.
  - The spec resolves or contradicts an open tension without a
    `## Design changes` entry that records the resolution and
    moves the tension file appropriately.
  - The spec introduces a new load-bearing noun without a
    `## Design changes` entry creating the concept file.
  - A `## Design changes` section exists but is incomplete (e.g.,
    references a concept mutation without saying which file or
    what changes) — execute-plan needs precise instructions.

  If `.ok-planner/design/concepts/` does not exist, skip this
  section.

  ## Spec quality checks

  After grounding, check the spec for:
  - Completeness (TODOs, placeholders, incomplete sections)
  - Internal consistency (contradictions, conflicting requirements)
  - Clarity (ambiguity that could cause wrong implementation)
  - YAGNI (unrequested features)

  Flag any PR, commit, branch, deployment, release, or rollout
  instructions — specs cover what to build, not how to ship it.
  Flag any indication that the work has been split across multiple
  specs.

  Only flag issues that would cause real problems during
  implementation.

  Report: Approved | Issues Found (with specifics)
```

Fix issues, re-review until clean. Each re-review runs the full grounding pass — do not assume the previous round was exhaustive. Cluster bias means missed grounding issues hide behind the ones the previous reviewer found. Then ask user to review the written spec.

## Transition

Once the spec is approved, invoke ok-planner:write-plan.
