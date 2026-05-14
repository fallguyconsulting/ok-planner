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

  ### Design log (if present)
  If `.ok-planner/design/concepts/` exists, consult it before
  forming findings:

  1. Read `.ok-planner/design/concepts.md` (the TOC).
  2. For files in scope, `rg '@concept:' <path>` for inline
     citations.
  3. Read `concepts/<slug>.md` for concepts surfaced by (1) or (2).

  Don't flag code that a concept documents as the intended shape.
  Cite the concept file when a finding genuinely contradicts one.
  Findings matching an open `tensions/<slug>.md` are duplicates —
  note the tension slug.

  If the spec has a `## Tensions resolved` section, verify the
  implementation actually applied each listed concept-file mutation
  and tension-file move (first-class plan tasks).

  **Annotate-on-consult (read-only variant):** When you consulted a
  concept and the load-bearing site has no `@concept:` annotation,
  note "consulted concept: <slug> (annotation missing at
  <file:line>)" in the finding so the fixer adds it.

  If `.ok-planner/design/concepts/` doesn't exist, ignore this
  section.

  ### Output Format

  For each spec requirement, report:
  - Implemented correctly
  - Missing (not implemented)
  - Incorrect (implemented but wrong)
  - Deviated (implemented differently than specified)

  Then list any additional issues found with file:line references.
  Do not categorize by severity. Every issue needs fixing.
```

5. If the reviewer found issues, invoke `ok-planner:review-cleanup` with the reviewer's full output. Do NOT process, filter, or triage the output yourself.
