---
name: review-work
description: "ONLY activated by explicit /review-work slash command. Never auto-triggered by conversation content."
---

# Review Uncommitted Work

Review all uncommitted changes (staged, unstaged, and untracked files). This is the "am I done?" check before committing.

For other review types, see: `/review-plan`, `/review-commits`, `/review-files`, `/review-feature`.

## Two independent cycles

`review-work` runs **two independent review/fix cycles**:

1. **Code review cycle** (always runs). The reviewer prompt below evaluates correctness, safety, state integrity, test coverage, and spec/plan conformance. Findings pass through `ok-planner:review-cleanup` and loop to clean.
2. **Design-doc compliance cycle** (runs only if `.ok-planner/design/concepts/` exists). A separate reviewer prompt audits the design docs against the concept self-containment rule. Findings pass through their own `ok-planner:review-cleanup` invocation and loop to clean.

The cycles are independent: findings do NOT mix, each has its own `review-cleanup` pass, each loops to clean independently. Run the code-review cycle first; once it reports clean, run the design-doc compliance cycle. Only report `review-work` clean to the orchestrator when BOTH cycles have reported clean. If either cycle hits the `review-cleanup` cap, surface its remaining issues grouped by cycle.

If `.ok-planner/design/concepts/` does not exist, skip the second cycle silently — the design docs are opt-in.

## Cycle 1 — Code review

## Reviewer Prompt

Dispatch a reviewer subagent:

```
Agent (general-purpose):
  ## Code Review

  ### Scope
  [Describe what to review based on the resolved scope — which files, diffs, or areas]

  ### How to find the changes

  **Working-tree first.** Run `git status` and `git diff` before
  forming any findings. Working-tree state is your subject — code
  modified or deleted in the working tree is in scope even if HEAD
  looks different. Don't form findings about code that exists in HEAD
  but has been deleted in the working tree. Don't omit code added in
  the working tree just because it's uncommitted.

  [Include the appropriate commands based on scope:]
  - Uncommitted: `git diff`, `git diff --staged`, `git status` for untracked files
  - Commit range: `git diff BASE..HEAD`
  - Plan: read the spec/plan, then read all files it touches
  - File/directory: read the files directly
  - Feature area: search for relevant files, read them
  - Submodules: if `git status` shows modified submodules, run `git diff --submodule=diff` to include submodule changes in the diff, or `git -C <submodule-path> diff` for each modified submodule. Submodule changes are part of the review scope — do not skip them.

  Read modified and new source files in their entirety for context.

  ### Review Focus
  - Correctness: bugs, edge cases, off-by-one errors
  - Safety: data loss, security issues, resource leaks, irreversible operations
  - State integrity: can we get stuck in a state? Double-execute? Skip steps?
  - Load-bearing properties upheld: name the properties the spec/plan depend on — durability, completeness, atomicity, ordering, idempotency, no-data-loss, "this record is canonical/authoritative," and the like — and verify the code actually guarantees each one. The dangerous failure is code that works on the happy path but silently trades such a property away for a local optimization (e.g. an audit write made asynchronous-and-droppable to protect request latency, quietly downgrading a record the spec called authoritative). For each property ask: does the implementation still guarantee it, or only under light load / the happy path? If the spec relied on a property the code no longer guarantees, that is an issue even when nothing appears "broken" — review the property, not just the line-by-line diff.
  - Test coverage: do tests verify real behavior? Any gaps?
  - Dead code, unused imports, stale comments
  - Anything that deviates from requirements
  - Pre-existing issues: if you find bugs, stale references, or inconsistencies in any file you read — even if they predate the current changes — report them. If an issue points to problems in files outside the initial scope, follow the trail and report those too.

  ### Source of truth
  The spec (linked from the plan's `**Spec:**` header, if any) and
  the plan itself are the source of truth for what the work was
  meant to accomplish. Form your judgments against the spec/plan,
  not against the design docs under `.ok-planner/design/` — the
  design docs as oracle are the concern of `review-holistic`
  (which uses them as the source of truth for whole-codebase
  convergence) and the read-only consultants (`brainstorm`,
  `refine-design`, `merge`).

  If the spec has a `## Design changes` section, open the affected
  files under `.ok-planner/design/` to verify each listed mutation
  landed correctly (concept file edits, tension file moves to
  `_resolved/`, new concept or tension files). That's not
  consulting the design docs as oracle — it's verifying that
  spec-directed work was carried out. If the spec doesn't address
  a question your review raises, raise it as an issue against the
  spec/plan, not against the design docs.

  ### Divergence report
  If the work was produced by `ok-planner:execute-plan`, a divergence
  report may exist alongside the plan (e.g.,
  `.ok-planner/plans/<plan>-divergences.md`). If present, read it. The
  divergence report is an auditor's record of where the implementation
  differs from what the plan literally said — useful context for
  understanding why the code looks the way it does, not a bypass of
  review. A divergence by itself is not an issue; form your findings
  from the code. A divergence that produces working, safe code is
  fine. A divergence that introduces a bug, regression, or invariant
  violation is an issue — flag it with file:line as you would any
  other. So is a divergence that silently trades away a property the
  spec or plan relied on (durability, completeness, atomicity,
  ordering, no-data-loss) — even when the code "works": flag it and
  name the property that was spent. And note that the auditor records
  consequential choices the plan left *unspecified* too; an
  unsettled-tradeoff divergence is one of those, so don't dismiss it
  just because the plan never named the property at stake.

  ### Output Format

  List every issue found with:
  - File:line reference
  - What's wrong
  - Why it matters
  - How to fix

  Do not categorize by severity. Every issue needs fixing.
  Report strengths briefly. Focus on problems.
```

## After the code-review cycle

If the reviewer found issues, invoke `ok-planner:review-cleanup` with the reviewer's full output. Do NOT process, filter, or triage the output yourself. Pass it through verbatim and let the cleanup skill drive the fix cycle.

Once `review-cleanup` reports clean (or hits its cap), proceed to the design-doc compliance cycle below.

## Cycle 2 — Design-doc compliance

This cycle is independent of the code review above. It exists so that the design docs under `.ok-planner/design/` stay in compliance with the concept self-containment rule (canonically stated in `ok-planner:discover-design`'s SKILL.md). Drift in those files isn't a code defect, so the code-review cycle wouldn't surface it; it needs its own reviewer and its own fix loop.

The cycle runs unconditionally when `.ok-planner/design/concepts/` exists — even if the current uncommitted change didn't touch any file under `.ok-planner/design/`. Drift can leak in indirectly (a code refactor renames a thing a concept body had cited; a prior plan's design-doc mutations slipped a path into the body). The "Fix Every Bug" rule applies: pre-existing violations are in scope.

If `.ok-planner/design/concepts/` does not exist, skip the entire cycle silently.

### Compliance Reviewer Prompt

Dispatch a reviewer subagent (separate from the code-review reviewer above):

```
Agent (general-purpose):
  ## Design-doc compliance review

  ### Your job

  Audit the project's design docs for compliance with the
  concept self-containment rule and the tension surface rule
  (both canonically stated in `ok-planner:discover-design`'s
  SKILL.md). Surface every violation as an issue; a fixer will
  rewrite. Do not triage. Pre-existing violations — in files
  the current uncommitted change did not touch — are still in
  scope.

  ### Scope

  In scope:
  - Every `.md` file directly under
    `.ok-planner/design/concepts/` (live concepts only — skip
    files under `concepts/_retired/`).
  - Every `.md` file directly under
    `.ok-planner/design/tensions/` (live tensions only — skip
    files under `tensions/_resolved/` and
    `tensions/_rejected/`).
  - `.ok-planner/design/concepts.md` (the auto-generated TOC).

  Out of scope (do NOT flag content here):
  - `.ok-planner/design/_discover/` — phase 1 scaffolding is
    allowed to cite code paths freely.
  - `.ok-planner/design/concepts/_retired/`,
    `tensions/_resolved/`, `tensions/_rejected/` — terminal
    state, historical record.
  - `.ok-planner/design/review-notes.md` and any dated
    rotations (`review-notes-YYYY-MM-DD.md`) — workflow
    scratch.

  ### Rules to enforce

  **Concept self-containment rule** (applies to every live
  `concepts/<slug>.md`):

  Concept body is self-contained. The design owns the
  definition; code references it via `@concept:` annotations.

  Disallowed in concept body:
  - File or directory paths in any form: `foo/bar.go`,
    `services/widget/`, `pkg:github.com/...`, bare URLs,
    `code:foo.go::Symbol`, "the code at X" pointers.
  - References to external documentation: `docs/...`,
    READMEs, CHANGELOG, sibling-repo paths.
  - Quoted code, quoted lint-config allowlists, quoted
    external prose.
  - "Owns / Does NOT own" sections that name code paths.
    (Boundaries is the in-vs-out section; it names neighbor
    concepts by slug.)

  Allowed in concept body:
  - Other concept slugs (`see also: claim-handle`,
    `concept:claim-handle`).
  - Annotation IDs the codebase uses (`@blessed-invariant: N`,
    `@agent-contract: X`).
  - Spec slugs in dated Notes entries
    (`spec:YYYY-MM-DD-<topic>`).
  - Dates.

  **Tension surface rule** (applies to every live
  `tensions/<slug>.md`):

  - `## What is muddy` and `## Evidence` — code-citation
    evidence is fine here. Don't flag citations.
  - `## Resolution candidates` — path-free. State changes at
    the concept level (which concept's Definition /
    Boundaries / Invariants would change), not the
    implementation level. Any file path, symbol citation, or
    external-doc reference here is a violation.

  **TOC consistency** (`concepts.md`):
  - Every TOC bullet's slug matches a live
    `concepts/<slug>.md` file. (Retired-only entries belong
    in the "Retired concepts" section, not the live list.)
  - Every live `concepts/<slug>.md` (non-retired) has a TOC
    entry.
  - One-sentence TOC definitions follow the same
    self-containment rule — no paths, no external-doc refs.

  **Cross-reference integrity**:
  - Every `see also: <slug>` and `concept:<slug>` referenced
    from a concept body resolves to a live concept file. A
    reference to a retired-only target is a violation —
    either repoint to the live successor or remove.
  - Every tension's `affects:` frontmatter slug resolves to
    a live concept.

  ### How to scan

  Walk every in-scope file. For each violation record:
  - File path
  - Line number or section heading
  - The offending text (quote it)
  - Which rule it violates
  - How to fix (rewrite as path-free prose, move the content
    to `_discover/`, file a tension if the concept genuinely
    can't say what it needs to without naming a path, etc.)

  ### Pre-existing issues

  Pre-existing drift that the current uncommitted change
  didn't introduce is still in scope. The "Fix Every Bug You
  Find" rule applies — surface every violation; the fixer
  will fix every one.

  ### Output format

  ```
  Status: Approved | Issues Found

  ## Findings

  (if Issues Found, one entry per violation:)

  ### <file>:<line-or-section> — <one-line summary>
  <Quoted offending text, which rule it violates, how to
  rewrite.>

  (if Approved:)

  (empty Findings section)
  ```

  ### Anti-padding

  - Don't flag content under `_discover/`, `_retired/`,
    `_resolved/`, `_rejected/`, or `review-notes*.md`.
  - Don't flag prose style. The rule is structural — which
    kinds of citations and sections are present — not whether
    the prose reads well.
  - Don't flag a concept for missing content the rule doesn't
    require.
  - Don't grade severity. Every violation is in scope; pass
    them all to the fixer.
```

### After the compliance reviewer

If the reviewer found issues, invoke `ok-planner:review-cleanup` with the reviewer's full output. Do NOT mix with code-review findings — this is a separate invocation with its own loop. Pass the compliance reviewer's output verbatim.

Once the compliance `review-cleanup` reports clean (or hits its cap), the cycle is done.

## Final report

When both cycles have reported clean, report `review-work` clean. If either cycle hit `review-cleanup`'s cap, surface remaining issues grouped by cycle (code-review remaining vs. compliance remaining); the user decides what to do.
