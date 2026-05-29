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

1. **Affirm layout** -- invoke `ok-planner:affirm` so `.ok-planner/specs/` exists before any writing starts.
2. **Establish the goal** -- if context from the conversation makes it clear what to brainstorm, proceed. Otherwise ask the user what they want to design. If invoked from `ok-planner:refine-design`, the goal has already been established in the conversation (a brief naming: the session's tension queue with picked resolution shapes, any tensions rejected during intake, affected concept files, code paths needing reconciliation, review-notes outcomes, and a proposed spec slug) — recognize that brief as the goal and proceed. Every item in the brief that touches the design docs must end up under `## Design changes` in the spec. If the brainstorm is initiated from one or more existing sketches under `.ok-planner/sketches/` — the user names one, or adopts one surfaced during context exploration — note them as the brainstorm's **source sketches**: the spec supersedes them, and they get archived once it is approved (see "Source sketches").
3. **Explore project context** -- check files, docs, recent commits, and (if it exists) the design docs under `.ok-planner/design/` to understand "how the project is now"
4. **Ask clarifying questions** -- one at a time, prefer multiple choice
5. **Propose 2-3 approaches** -- with trade-offs and your recommendation
6. **Present design** -- scaled to complexity, get user approval section by section. Surface any design-doc impacts (concept additions/mutations, tension resolutions or new tensions) as part of the discussion; they go into the spec, not into the design docs directly.
7. **Write spec** -- save to `.ok-planner/specs/YYYY-MM-DD-<topic>-design.md`. If the work touches the design docs, include a `## Design changes` section enumerating the mutations execute-plan will apply.
8. **Spec review** -- dispatch reviewer subagent, fix issues, re-review until clean
9. **User reviews spec** -- ask user to review before proceeding
10. **Archive source sketches** -- if the brainstorm had source sketches: capture any deferred content in a new sketch, then move each source sketch to `.ok-planner/history/sketches/` (see "Source sketches"). Skip if there were none.
11. **Transition** -- invoke ok-planner:write-plan

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
- Sequencing and task ordering are `write-plan`'s concern — defer if it comes up, unless the ordering choice is load-bearing for the design (it enables an invariant, rules out a footgun, picks between materially different futures), in which case it's a design decision and stays in the spec.

## Presenting the Design

- Scale each section to its complexity
- Ask after each section whether it looks right
- Cover: architecture, components, data flow, error handling, testing
- YAGNI ruthlessly

## Surface every tradeoff — never resolve one silently

Many design decisions trade one desirable property against
another: correctness vs. speed, completeness vs. latency,
durability vs. cost, robustness vs. simplicity, atomicity vs.
throughput. When a decision has this shape, **name the tradeoff
and let the user decide it.** Do not pick a side on their behalf,
and do not let a default feel so obvious that you skip surfacing
it — "obviously we'd make this async so it doesn't add latency" is
exactly the call that silently spends a load-bearing property (a
complete, durable record) to buy a local one (request latency).

The failure mode this guards against is not "the agent chose
wrong." It is "the agent traded away a load-bearing property to
optimize a local concern, never recognized it as a tradeoff, and
so never surfaced it." The standard is: default to rigor, and
surface anything that strays from it.

In practice:
- When a choice pits two good properties against each other, stop
  and put both sides to the user — in prose, with your
  recommendation — one question at a time, like any other
  clarifying question.
- A resolved tradeoff is a design decision: record it in the spec
  (and under `## Design changes` if it changes a concept's
  invariants or boundaries).
- A tradeoff the user chooses to leave open is a tension: capture
  it as a `## Design changes` tension entry (see "Tension
  impacts") — not a silent default.
- The dangerous tradeoff is the one you don't see. When a choice
  protects a local metric (latency, code size, throughput), ask
  what global property it might be spending — completeness,
  durability, ordering, atomicity — and surface the exchange.

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

**Concept self-containment in spec-driven mutations.** When a
`## Design changes` bullet mutates a concept's body — new or
rewritten Definition / Purpose / Boundaries / Invariants — the
new body must follow the concept self-containment rule from
`ok-planner:discover-design`'s SKILL.md ("Concept
self-containment rule"). TL;DR: concept body has no file paths,
no `code:`/`pkg:` citations, no external-doc references
(`docs/...`, READMEs, sibling-repo paths), no quoted code or
lint-config allowlists, no "Owns / Does NOT own" sections that
name code paths. Allowed citations are other concept slugs,
annotation IDs, spec slugs in dated Notes entries, and dates.

State the new section text directly in the bullet; do not phrase
the mutation as "see `foo.go`" or "matches `docs/...`" or "the
canonical impl lives at `pkg:...`". If a tension's resolution
shape can't be stated as concept-doc + code-discipline changes
without naming a path, the resolution isn't ready — sharpen it
before writing the spec.

The same rule applies to tension files the spec creates or
modifies: `## Resolution candidates` sections in tensions are
path-free; `## What is muddy` and `## Evidence` can cite code
as evidence. See the "Tension surface rule" in the
extractor prompt.

## Writing the spec and reviewing it

**Before writing the spec, ground its citations against the actual source.** For every code surface the spec is going to name — files, functions, structs, tables, columns, proto messages, line numbers — read the underlying file first and copy the names verbatim. Do not write a citation from inference or from conversation context. Hallucinated citations cluster: a single wrong file path or function name in the first draft tends to spawn a wrong path or name in each round of fixes (cluster bias on the fix side, mirroring the reviewer's). Specs that touch many code surfaces and rely on guessed citations need many review rounds to converge; specs that ground every citation up-front converge in one or two. The cost of the up-front grounding pass is much smaller than the cost of the fix-review loop it prevents.

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
  - A `## Design changes` bullet mutates concept body text but
    the new text violates the concept self-containment rule
    (see `ok-planner:discover-design`'s SKILL.md, "Concept
    self-containment rule"): cites a file path, names an
    external doc, quotes code or a lint-config allowlist, or
    introduces an "Owns / Does NOT own" section that cites
    paths. Flag the offending bullet and the offending
    citation; the fix is to rewrite the new body as path-free
    prose.
  - A `## Design changes` bullet creates or modifies a tension
    whose `## Resolution candidates` section cites a file path,
    symbol, or external doc. Resolutions become spec
    instructions and live forward in time — they must be
    path-free (see the "Tension surface rule"). Code-citation
    evidence is fine in `## What is muddy` / `## Evidence`,
    not in `## Resolution candidates`.

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

## Source sketches

A brainstorm is often initiated from an existing sketch under
`.ok-planner/sketches/` — the user points at one and says "design
this," or one surfaces during context exploration (step 3) and the
user adopts it as the thing to brainstorm. There can be more than
one. Those are the brainstorm's **source sketches**: the spec it
produces supersedes them.

A sketch read only as adjacent background — context the brainstorm
does not consume — is **not** a source sketch and stays where it
is. The test: a sketch is a source sketch only if this brainstorm
dispositions its content (below). When it's unclear whether a
sketch the user mentioned is a source or just background, ask.

There is no other mechanism that moves sketches out of
`.ok-planner/sketches/`. Brainstorm is where a sketch is consumed
into a spec, so brainstorm is where its source sketches are
archived.

### Dispositioning sketch content

By the time the spec is approved, every piece of a source sketch
has landed in exactly one of three places:

- **In scope** — captured in the spec. The normal case.
- **Declined or reframed** — dropped. The user decided not to do
  it, or the brainstorm folded it into something else already in
  the spec. Nothing carries forward.
- **Deferred** — the user explicitly chose to do it later: not
  declined, not reframed, *deferred*. Deferred work must not
  vanish when the source sketch is archived, so it is captured in
  a new sketch (below).

Defer-vs-decline is the user's call, made during the dialogue — do
not unilaterally defer. When sketch content falls out of the
brainstorm's scope, surface it and let the user say decline /
reframe / defer.

### Archiving (after the spec is approved)

Once the user has approved the spec (step 9), before transitioning
to write-plan:

1. **Capture deferred work.** If any source-sketch content was
   deferred, write a new sketch to
   `.ok-planner/sketches/YYYY-MM-DD-<topic>-sketch.md` capturing
   the remaining work, using the sketch template from
   `ok-planner:sketch`'s SKILL.md. Give it a topic slug that
   reflects the deferred work, not the original sketch. (Split
   into more than one sketch only if the deferred work spans
   clearly separate topics.)
2. **Archive the originals.** Move each source sketch to
   `.ok-planner/history/sketches/`, preserving the filename
   (`git mv` if tracked, else `mv`).
3. **Report** what was archived and the path of any deferred-work
   sketch you wrote.

Do not ask permission to archive: it is non-destructive (the
sketch stays readable under `history/`) and is the documented
completion behavior, the same way `execute-plan` archives a plan
when it finishes. Just do it and report.

## Transition

Once the spec is approved, invoke ok-planner:write-plan.
