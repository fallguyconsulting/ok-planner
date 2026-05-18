---
name: review-plan
description: "ONLY activated by explicit /review-plan slash command. Never auto-triggered by conversation content."
---

# Review Plan Implementation

Help the user select a plan to review against, then dispatch a reviewer.

## Steps

1. List spec and plan files in reverse chronological order:
   - Primary location: `.ok-planner/specs/` and `.ok-planner/plans/` (current ok-planner artifact layout)
   - Also include `.ok-planner/history/specs/` and `.ok-planner/history/plans/` so completed work can be re-reviewed
   - Fallback: older projects may still have artifacts under `docs/specs/` or `docs/plans/` — include those if present
   - Present as a numbered multiple-choice list, newest first
   - Include the filename and first line (title) of each

2. Ask the user which plan/spec to review against.

3. Once selected, read the spec/plan to understand what was intended.

4. Dispatch a reviewer subagent:

```
Agent (general-purpose):
  ## Review Plan Implementation

  ### Spec/Plan
  [Path to the selected spec/plan. Read it in full.]

  ### Your Job
  Read the spec/plan, then read every file it references — including files inside git submodules. For each requirement:
  - Is it implemented?
  - Is the implementation correct?
  - Are there gaps, edge cases, or deviations?

  Also check for:
  - Code that was added but isn't in the spec (scope creep)
  - Tests that don't actually verify the spec requirements

  ### Pre-existing issues
  If you find bugs, stale references, or inconsistencies in any file you read — even if they predate the current work — report them. "It was already broken" is not a reason to skip it. If an issue in one file points to problems in files outside the spec's scope, follow the trail and report those too.

  ### Source of truth
  The spec/plan is the source of truth for this review. Form your
  judgments against the spec/plan, not against the design docs
  under `.ok-planner/design/` — the design docs as oracle are the
  concern of `review-holistic` (read-only oracle consumer) and
  the read-only consultants (`brainstorm`, `refine-design`,
  `merge`).

  If the spec has a `## Design changes` section, open the
  affected files under `.ok-planner/design/` to verify each
  listed mutation landed correctly (concept edits, tension file
  moves to `_resolved/`, new concept or tension files). That's
  not consulting the design docs as oracle — it's verifying
  spec-directed work. If your review raises a question the
  spec/plan doesn't address, raise it as an issue against the
  spec/plan, not against the design docs.

  ### Divergence report (if present)
  If the plan was executed by `ok-planner:execute-plan`, a
  divergence report may exist at
  `.ok-planner/plans/<plan>-divergences.md` (or its archived
  equivalent under `history/plans/`). If present, read it for
  context on creative choices the implementer made. A divergence
  by itself is not an issue — form your findings from the code.
  A divergence that produces working, safe code is fine; one that
  introduces a bug, regression, or invariant violation is an
  issue.

  ### Output Format

  For each spec requirement, report:
  - **Implemented correctly** — the requirement is met, whether
    the code follows the plan literally or arrived at the same
    business outcome by a different shape.
  - **Missing** — the requirement was not implemented.
  - **Broken** — the requirement was attempted but the
    implementation has a bug, regression, or invariant violation.

  Deviation from the plan's literal wording is NOT a finding by
  itself — if the spec's intent is met, it's correct. Only flag
  deviations that produce broken code.

  Then list any additional issues found with file:line references.
  Do not categorize by severity. Every issue needs fixing.
```

5. If the reviewer found issues, invoke `ok-planner:review-cleanup` with the reviewer's full output. Do NOT process, filter, or triage the output yourself.
