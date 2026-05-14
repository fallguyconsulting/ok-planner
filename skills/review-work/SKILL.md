---
name: review-work
description: "ONLY activated by explicit /review-work slash command. Never auto-triggered by conversation content."
---

# Review Uncommitted Work

Review all uncommitted changes (staged, unstaged, and untracked files). This is the "am I done?" check before committing.

For other review types, see: `/review-plan`, `/review-commits`, `/review-files`, `/review-feature`.

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
  - Test coverage: do tests verify real behavior? Any gaps?
  - Dead code, unused imports, stale comments
  - Anything that deviates from requirements
  - Pre-existing issues: if you find bugs, stale references, or inconsistencies in any file you read — even if they predate the current changes — report them. If an issue points to problems in files outside the initial scope, follow the trail and report those too.

  ### Design log (if present)
  If `.ok-planner/design/concepts/` exists, consult it before
  forming findings:

  1. Read `.ok-planner/design/concepts.md` (the auto-generated
     TOC — small, one-shot).
  2. For files in scope, grep for `@concept:` annotations
     (`rg '@concept:' <path>`) — each marks a load-bearing site.
  3. Read `concepts/<slug>.md` in full for concepts surfaced by
     (1) or (2).

  If a concept's stated boundary or invariant says the code is
  correct as written, do NOT flag — the documented design is the
  oracle, not your intuition. If a finding genuinely contradicts a
  concept, cite the concept file in the finding alongside
  file:line. Findings that match an open entry in
  `.ok-planner/design/tensions/` are duplicates — note the tension
  slug rather than re-litigating.

  **Annotate-on-consult (read-only variant):** When you consulted a
  concept to form a finding AND the load-bearing site has no
  `@concept:` annotation, note "consulted concept: <slug>
  (annotation missing at <file:line>)" in the finding. The fixer
  dispatched by `review-cleanup` will leave the annotation as part
  of the fix.

  If `.ok-planner/design/concepts/` doesn't exist, ignore this
  section.

  ### Implementation Notes File
  If the work was produced by `ok-planner:execute-plan`, an implementation notes file may exist alongside the plan (e.g., `.ok-planner/plans/<plan>-notes.md`). If present, read it. Deviations logged there are part of the review scope — for each entry, verify that the implementation actually matches what the note claims, and flag any deviation that produces an incorrect or unsafe result as an issue. Notes are not a bypass of review; they are data for it.

  ### Output Format

  List every issue found with:
  - File:line reference
  - What's wrong
  - Why it matters
  - How to fix

  Do not categorize by severity. Every issue needs fixing.
  Report strengths briefly. Focus on problems.
```

## After Review

If the reviewer found issues, invoke `ok-planner:review-cleanup` with the reviewer's full output. Do NOT process, filter, or triage the output yourself. Pass it through verbatim and let the cleanup skill drive the fix cycle.
