---
name: review-holistic
description: "ONLY activated by explicit /review-holistic slash command. Never auto-triggered by conversation content."
---

# Review Holistic

Codebase-wide review aimed at convergence, not at the most recent change. The
review subject is the whole codebase, regardless of what was recently
modified or what plan/spec drove the changes. The reviewer is directed to
look for both real defects and extraneous accretion — code, tests, features,
abstractions, and documentation that don't earn their keep.

## When to use

- After a large delivery (multi-dispatch implementation, major feature
  landing) when the goal is to drive the whole codebase to a consistent,
  parsimonious state — not just to confirm the recent changes are correct.
- When prior `review-work` / `review-plan` cycles have plateaued and
  finding-counts aren't trending toward zero. Different review prompts
  steer the reviewer's attention; this one steers toward
  whole-codebase quality and parsimony.
- When the user wants a "convergence pass" — running review-fix-review
  until a cycle finds zero issues across all four classes below.

For other review types: `/review-work` (uncommitted only),
`/review-plan` (against spec/plan), `/review-commits` (commit range),
`/review-files` (specific files), `/review-feature` (feature area).

## Convergence is the stopping criterion

This review is intended to run repeatedly under `review-cleanup` until a
single cycle returns zero findings across all four classes. There is no
turn budget; convergence is the criterion. If `review-cleanup`'s built-in
3-cycle limit triggers and findings remain, the user decides whether to
continue.

Adding code in response to a finding (e.g., closing a real gap) is correct
and not a sign of non-convergence. Non-convergence looks like the codebase
accreting extraneous features, tests, or abstractions — code added that
doesn't earn its keep. The reviewer prompt directs attention at exactly
this distinction.

## Reviewer Prompt

Dispatch a reviewer subagent:

```
Agent (general-purpose):
  ## Codebase Review (holistic)

  ### Scope

  The whole codebase. Not the recent change, not the plan that drove it,
  not the dispatch boundary. Treat the entire repository as the review
  subject. Any plan or spec is informational — neither a fence around
  what to look at nor a checklist of what to verify.

  ### Your Job

  Look for FIVE classes of finding. The first four are equally
  important; the fifth (Questions) is for plausibly-design judgment
  calls that you'd want a human's read on before acting. Don't rank
  within a class. Don't grade by severity. List flat findings under
  each class.

  #### 1. Real issues (anywhere in the codebase)

  Bugs, inconsistencies, missing-but-needed behavior, regressions, broken
  invariants, race conditions, incorrect comments, stale references, error
  paths that swallow failure silently. The plan/dispatch boundary is
  irrelevant. An issue in code that wasn't touched by recent work is still
  an issue. If recent code rendered older code stale or wrong, that older
  code is now in scope.

  Pay particular attention to:
  - Interfaces declared with no production implementation
  - Fields populated at one site but never read (or read at a site that
    doesn't expect them)
  - Helpers/services constructed but never invoked at a real call site
  - "Wired up" surface that doesn't reach a runtime call site
  - Doc comments that describe a different behavior than the code

  #### 2. Code that doesn't earn its keep

  What in this codebase is extraneous, redundant, or exists for its own
  sake?

  - Tests that don't exercise real behavior (assert "no error returned"
    only; assertions that would pass even if the code under test were a
    no-op; tests that verify implementation details that don't matter).
  - Helpers / abstractions / interfaces with one implementation and one
    caller — flatten them.
  - Factory functions that just allocate a struct.
  - "Manager" classes, abstractions added for hypothetical future
    flexibility, premature configuration knobs, parameters that are
    accepted and immediately discarded.
  - Features without load-bearing use; code paths that exist but no
    actual call site exercises; config options nobody sets.
  - Dead-by-implication code. Old fallbacks that the new primary path
    makes irrelevant. Backwards-compat shims for a v1 that hasn't
    shipped, when the project rules permit breaking changes.
  - Documentation that's words to fill space rather than guidance a
    reader needs.

  #### 3. Inconsistencies across the system

  Where does the codebase contradict itself or its own conventions?

  - New code that diverges in style/pattern from established old code.
  - Old code that's now stale because newer code uses a different pattern
    — and the old call sites were missed.
  - Two ways to do the same thing where one would be enough; duplicated
    helpers without `@source` annotation.
  - Naming/structure conventions that some sites follow and others
    violate.
  - Documentation that says one thing while the implementation does
    another.
  - Project-wide conventions (cold-read, blessed-invariant annotations,
    explicit code over magic, max import depth, per-feature
    organization) violated.

  #### 4. Churn / gratuitous changes

  What was changed/added that didn't need to be?

  - Rewrites of working code, gratuitous renames, refactors that produced
    no real improvement.
  - Docs added because "we should have docs" rather than because a reader
    needs them.
  - Features added because they were on a plan rather than because they
    earn their keep.
  - Config or env-var constants defined and then unused.
  - Placeholder files that exist to host a comment about where the real
    code lives.
  - Variable shadowing of language builtins introduced by recent
    additions.

  #### 5. Questions (plausibly intentional)

  Findings where you suspect something is wrong but it could plausibly
  be a deliberate design choice you don't have context for. Surface
  them so the user sees them, but they are NOT for auto-fix.

  Use this class for:
  - "Could be inlined" / "could be flattened" / "could be unified"
    judgment calls where the helper / abstraction's justification
    depends on intent rather than provable wrongness.
  - Extracted-too-early concerns where the extraction might document
    an authorial intent you can't see.
  - Naming-convention concerns where the convention itself is
    debatable.
  - Dead-looking code that might document a deliberate negative
    choice (the constant exists so a test pins the rejection, the
    interface is a forward-compat seam, etc.).
  - Findings where you read the design-log Notes section and the
    Notes neither support nor refute your concern.

  The bar: "if I had to defend this in front of someone who designed
  it, what would they likely say?" If a plausible defense exists, the
  finding goes here. If no plausible defense exists, it's a real
  finding in classes 1-4.

  ### Project conventions

  Read the repo root `CLAUDE.md` and any `.claude/rules/` files before
  forming findings. The repo's stated conventions are the bar; deviations
  from them are findings.

  Pre-v1 projects with explicit "break freely" rules favor net-removal
  fixes over backwards-compat shims. If the repo says break freely,
  surface backwards-compat shims as Class-2 findings.

  ### Implementation notes and design log (required reading)

  Before forming any findings, you MUST read:

  - Every `*-notes.md` file under `.ok-planner/plans/` in full.
    These record deliberate deviations from each plan as it was
    executed.
  - **`.ok-planner/design/concepts.md` (if it exists)** — the
    auto-generated TOC of the project's canonical concept
    catalog. Holistic review is whole-codebase scope, so the TOC
    is your starting map of what's load-bearing.
  - **`@concept:` annotations across the codebase** — run
    `rg '@concept:'` to surface every inline citation. Each tells
    you a load-bearing site for a specific concept.
  - **Full `.ok-planner/design/concepts/<slug>.md` files** — read
    every concept the TOC and annotations surface. For holistic
    review, expect this to be most or all of them; the catalog is
    the design oracle.
  - **Every tension file under `.ok-planner/design/tensions/`** (if
    the directory exists). These document the design's known
    muddiness — code shape that matches an open tension is a
    duplicate of that tension, not a fresh finding.

  These artifacts document deliberate design choices and known open
  questions. They exist precisely so you can distinguish a defect
  from an intentional decision you don't have context for, and an
  original finding from a re-litigation of an already-cataloged
  tension.

  For any finding you're considering that overlaps a documented
  concept boundary or invariant: you MUST explicitly explain why the
  documented concept is wrong before flagging it as a code issue.
  "I don't see why this is shaped this way" is not enough — read the
  concept file and only flag if you can quote the specific
  Boundaries/Invariants/Purpose line that is itself flawed.

  A finding that disagrees with a documented concept without
  addressing the concept itself is noise. If you find documented
  concept material that is genuinely wrong, that's a Class-1 finding
  against the concept file, not a finding against the code.

  For any finding that overlaps an open tension entry: do NOT
  re-litigate. Note the tension slug alongside the finding and
  move on — the tension is the artifact of record.

  Conversely, do NOT flag a deviation from a concept just because
  it's a deviation. The artifacts are not off-limits to flag —
  they're a filter for distinguishing real issues from "I'd have
  done this differently."

  If `.ok-planner/design/concepts/` or `tensions/` does not exist,
  this project hasn't bootstrapped a design log. Form findings
  normally; do not propose creating the directories as a side
  effect.

  **Annotate-on-consult (read-only variant).** When you consulted a
  concept to form a finding AND the load-bearing site has no
  `@concept:` annotation, note "consulted concept: <slug>
  (annotation missing at <file:line>)" in the finding output. The
  fixer dispatched by `review-cleanup` will leave the annotation as
  part of the fix.

  ### What NOT to do

  - Don't categorize by severity. Every finding goes in one of the
    five classes; within a class, list flat with no sub-buckets.
  - Don't gate findings on "was this part of the recent dispatch."
    Anywhere in the codebase is in scope.
  - Don't suggest fixes that add more code unless they net-reduce
    complexity. Prefer reductions.
  - Don't rank or grade — list everything you find.
  - Don't pad. If a class has no findings, write `(none found)`.
  - Don't write a "summary" or "verdict" at the end. The findings list
    is the artifact.
  - Don't put findings in class 5 to soften them. Class 5 is for
    plausibly-design choices, not for findings you're unsure about.
    If you're unsure whether something is wrong, do more reading
    until you're sure, then put it in the right class.
  - Don't put findings in classes 1-4 if a concept file
    (`.ok-planner/design/concepts/<slug>.md`) documents the shape
    as deliberate. If the concept itself is wrong, your finding is
    "the concept at concepts/<slug>.md is wrong" — Class 1 against
    the concept file, not Class 1 against the code.
  - Don't re-file a finding that matches an open tension entry
    (`.ok-planner/design/tensions/<slug>.md`). Note the tension
    slug alongside the finding and move on; the tension is the
    artifact of record.

  ### How to find things

  **Working-tree first.** Before forming any findings, run `git status`
  and `git diff` to confirm what is modified relative to HEAD. The
  working tree is your subject — code that has been deleted in the
  working tree is not a finding even if it appears in HEAD. Code added
  in the working tree is in scope even though it's uncommitted. A
  finding pointing at a HEAD-only artifact that the working tree has
  already removed is noise.

  Read the codebase top-down. Start with the entry points (`cmd/*`,
  `bin/*`, `main.*`) and walk the import graph. For each layer, read
  the top-level files and follow into the components.

  Skim test directories. Are these tests doing real work or exercising
  trivia?

  Read at least the top of every executable/binary's main file to confirm
  what's actually shipping.

  Read every interface declaration in shared infrastructure packages.
  Confirm each interface has a production implementation, not just a
  test fake.

  Browse the docs. For each page, ask: "is this useful to a reader, or
  is it words to be words?"

  Use grep aggressively to verify wiring claims:
  - `grep -rn "<InterfaceName>" --include="*.go"` to find production
    implementations.
  - `grep -rn "<HelperName>" --include="*.go" | grep -v _test.go` to find
    non-test call sites.
  - Verify any "is wired" claim by tracing from a constructor or main
    function to a runtime call site.

  ### Output Format

  ```
  ## 1. Real issues

  - file:line — what's wrong (1-2 sentences). If non-obvious, why it matters.
  - ...

  ## 2. Code that doesn't earn its keep

  - file:line — what's extraneous and why it should be removed/flattened.
  - ...

  ## 3. Inconsistencies

  - file:line — what contradicts what.
  - ...

  ## 4. Churn / gratuitous changes

  - file:line — what should not have been added/changed.
  - ...

  ## 5. Questions (plausibly intentional)

  - file:line — what you noticed, the plausible design defense you
    can imagine, and what would tip the balance toward "real issue."
  - ...
  ```

  Within each class, list flat. No sub-severity, no per-component
  sub-buckets. Concrete file:line references where possible.

  If a class is empty, write `(none found)` — don't pad.

  After the five classes, you MAY briefly list strengths (3-5 lines max).
  Optional. Skip if nothing notable.
```

## After Review

Split the reviewer's output by class:

- **Classes 1-4 (real issues, doesn't-earn-keep, inconsistencies,
  churn).** If any are present, invoke `ok-planner:review-cleanup`
  with ONLY classes 1-4. Do NOT process, filter, summarize, triage,
  or rebut findings yourself. Do NOT include class 5 in the
  review-cleanup input — the fixer treats input as
  fix-mechanically; class-5 findings need human judgment first.
- **Class 5 (Questions).** Surface verbatim to the user. Do not
  invoke review-cleanup on these. The user walks each one and
  decides whether to fix, defer, or document the choice in the
  design log. After user input, any items the user routed to "fix"
  go to a follow-up review-cleanup pass with explicit fix
  directives; items the user routed to "design choice" become
  candidates for `.ok-planner/design/` entries.

If both classes are present, run review-cleanup on classes 1-4 to
completion FIRST, then walk the class-5 questions with the user. This
ordering keeps the auto-fix surface clean and the human-judgment
surface sequenced after the mechanical work settles.

`review-cleanup`'s built-in fixer is the standard fix-everything
subagent — it works the same for findings from this review as for any
other. The only difference is that this review's prompt steers the
reviewer toward parsimony and whole-codebase scope; the fix loop is
unchanged.

For the verification re-review at the end of each fix cycle,
`review-cleanup` will direct the orchestrator to re-dispatch the reviewer.
Use the SAME prompt as defined above — not a generic review-work prompt —
so convergence is measured against the same lens that surfaced the issues.
