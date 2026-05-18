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
  other.

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
