---
name: merge
description: "ONLY activated by explicit /merge slash command. Never auto-triggered by conversation content."
---

# Merge: Triage a Merge or Rebase

Two-stage review of a recently-merged-in change set. Stage 1
surfaces design-doc drift introduced by the merge — mechanical
issues, drift, semantic conflicts — as findings the user can feed
into a follow-up brainstorm / refine-design session that will
produce a spec. Stage 2 runs a code review of the merge diff via
`ok-planner:review-commits`.

If the project has no design docs (`.ok-planner/design/concepts/`
does not exist), stage 1 is skipped silently and `/merge` is
effectively a wrapper around `/review-commits` over the merge
range.

## Why this exists

A merge or rebase can introduce two kinds of inconsistency:

- **Design-doc inconsistencies** — two branches edited the same
  concept incompatibly, or code that just landed contradicts a
  concept's documented Boundaries / Invariants. The design docs
  are a source of truth with the same weight as code; like code,
  they only change through plan execution. `/merge` does not
  patch the design docs directly. It surfaces findings; the user
  takes them through `/brainstorm` or `/refine-design` to
  produce a reconciliation spec, which then flows through
  write-plan and execute-plan as usual.
- **Code issues** — the usual review surface, applied to the
  merge diff.

## When to invoke

Right after a git merge or rebase that touched code. Optionally before
a merge as a dry-run by checking out the merge base and running with the
prospective tip as the comparison ref.

## Inputs

- `.ok-planner/design/` — the current design docs (post-merge), if present.
- The merge diff: `git diff <base>..HEAD`, where `<base>` defaults to
  `HEAD^1` if HEAD is a merge commit, else `main`. Override with the
  skill's optional argument: `/merge main`, `/merge HEAD~5`, etc.

## Process

1. **Resolve the base ref.** If HEAD is a merge commit and no
   argument given, use `HEAD^1`. Otherwise default to `main`.
   Allow the user to override.

2. **Stage 1 — Surface design-doc findings.** If
   `.ok-planner/design/concepts/` does not exist, skip this stage
   silently — the design docs are opt-in. Otherwise:

   a. Dispatch the design-doc triage subagent (prompt below) to
      surface findings in three classes:
      - **Mechanical** — slug collisions, malformed templates,
        broken `@concept:` annotations.
      - **Drift** — a concept's Boundaries / Invariants now
        contradicts code that just landed; or `@concept:`
        annotations point at sites that have moved or no longer
        enforce what the concept claims.
      - **Semantic conflicts** — two branches edited the same
        concept's Definition / Purpose / Boundaries / Invariants
        incompatibly, or added concept files for arguably the
        same noun under different slugs.

   b. Save the findings to
      `.ok-planner/_merge-findings-<date>.md` as a structured
      markdown report (the subagent prompt below specifies the
      format). The file lives at the `.ok-planner/` root, not
      under `design/`, because `/merge` is read-only against the
      design docs. Do NOT modify any file under
      `.ok-planner/design/` to apply the findings — design docs
      change only through plan execution.

   c. Walk the findings with the user briefly:
      - Confirm each finding makes sense (drop spurious ones).
      - For each kept finding, suggest the natural next step:
        * **Mechanical** → small targeted spec via `/brainstorm`,
          with a `## Design changes` section listing the fixes.
        * **Drift** → small spec via `/brainstorm` with
          `## Design changes` capturing the Notes append (and any
          concept-edit that needs to ride with code changes that
          would land separately).
        * **Semantic conflicts** → suggest `/refine-design` after
          the user has at least one of the conflicting positions
          captured as a tension; or a follow-up `/brainstorm` if
          the user has already decided on a resolution.

      The user decides when to act. `/merge` does not invoke
      `/brainstorm` or `/refine-design` automatically.

   Single pass. No fix loop. No writes to the design docs.

3. **Stage 2 — Code review.** Invoke `ok-planner:review-commits`
   with the range `<base>..HEAD` as its skill argument so its
   interactive range-picker is bypassed. Let it run to completion
   via review-cleanup.

4. **End-of-run summary.** Report stage 1's findings counts and
   the path to the `.ok-planner/_merge-findings-<date>.md` report;
   report stage 2's findings + cleanup status. Remind the user that
   design-doc findings need a follow-up spec (via `/brainstorm`
   or `/refine-design`) to actually fix anything — `/merge` only
   surfaced them.

## Design-doc Triage Subagent Prompt

```
Agent (general-purpose):
  ## Triage Design Docs Against Merge

  ### Your job

  A merge or rebase has just landed. The design docs at
  `.ok-planner/design/` may now be inconsistent with themselves,
  with the codebase, or both. Surface the inconsistencies in three
  classes of finding for the orchestrator to walk with the user.
  **Do NOT modify any file under `.ok-planner/design/`** — design
  docs change only through plan execution. Your job is to draft
  findings, not apply them.

  ### Inputs

  - Current design docs: read every file under
    `.ok-planner/design/concepts/` and `.ok-planner/design/tensions/`.
    Read `.ok-planner/design/concepts.md` (the TOC) for orientation.
  - Merge diff: `git diff <BASE>..HEAD` where BASE was supplied as
    `<BASE_REF>`. Read the diff in full; identify what code changed.
  - Optionally: `git log <BASE>..HEAD --oneline` for commit-level
    structure.

  Do NOT read existing docs (READMEs, narrative under `docs/`, etc.).
  The design docs are the authoritative record of intent for this
  skill's purposes.

  ### Three classes of finding

  #### 1. Mechanical

  Trivial issues an agent can fix without judgment:
  - Slug collisions — two branches both added `concepts/<slug>.md` or
    `tensions/<slug>.md` with the same slug but different content.
  - Malformed templates — a concept or tension file missing required
    sections per `discover-design`'s templates.
  - Broken `@concept:` annotations — `rg '@concept:'` across the merge
    diff turns up annotations whose slug doesn't exist in `concepts/`.

  For each, propose a specific resolution: rename file A to a new
  slug; fill in the missing section; remove the broken annotation or
  correct the slug.

  #### 2. Drift

  A concept's prescriptive content now contradicts code that just
  landed:
  - Boundaries says X is in concept A, but the merge diff moved X
    elsewhere.
  - Invariants claims a property the post-merge code no longer holds.
  - `@concept:` annotations point at sites that have moved or no
    longer enforce what the concept claims.
  - **Citation-rot drift**: concept body cites a file path,
    symbol, or external doc that the merge moved, renamed, or
    removed. Per the concept self-containment rule (see
    `ok-planner:discover-design`'s SKILL.md), concept body
    should not contain such citations at all — the rot finding
    is also a self-containment finding.

  For each drift finding, draft a proposed Notes append for the
  affected concept file:
  - Concept file path
  - Specific evidence file:line where the drift shows up
  - One-line description of the observation
  - Proposed Notes append text (one or two lines, dated, citing the
    merge)

  Per the design docs' format-durability rules, do NOT propose
  rewrites of Definition / Purpose / Boundaries / Invariants in
  the merge findings. Notes appends are the natural shape for
  drift findings; bigger rewrites need a `/refine-design` session
  to produce a proper spec.

  **For citation-rot drift specifically:** propose REMOVING the
  offending citation from the concept body, not repointing it
  to the new path. Repointing perpetuates the drift problem —
  the next refactor will rot it again. If the surface the
  citation pointed at is genuinely load-bearing for the concept
  (the concept can't explain itself without naming the thing),
  that's information for the user to take through
  `/refine-design`: either sharpen the concept's Boundaries to
  state the load-bearing property at the concept level, or file
  a tension. The proposed Notes append should say so.

  #### 3. Semantic conflicts

  Two branches that both edited the same concept's prescriptive
  content incompatibly, or added concept files for arguably the same
  noun under different slugs. Detect by:
  - Concept files where both branches' diffs touched Definition /
    Purpose / Boundaries / Invariants.
  - Boundaries that now contradict each other or overlap (e.g., both
    A and B claim the same responsibility).
  - Concept slugs that are clear synonyms or near-synonyms of an
    existing concept.

  For each semantic conflict, draft a `tensions/<slug>.md` entry per
  `discover-design`'s tension template:
  - A short slug naming the area of disagreement.
  - The competing positions, summarized one each.
  - The concept files implicated.
  - The merge commit / range as the trigger.

  Do NOT pick a winner. The tension exists; capturing it is the
  point.

  ### Output format

  ```
  ## 1. Mechanical

  - <file-or-area> — <what's wrong> — <proposed fix>
  - ...

  ## 2. Drift

  - <concept-file>
    Evidence: <file:line>
    Observation: <one line>
    Proposed Notes append: <one or two lines, dated, citing the merge>
  - ...

  ## 3. Semantic conflicts

  - Proposed tension slug: <slug>
    Concept(s) implicated: <list>
    Position A: <one line>
    Position B: <one line>
    Drafted tension file:
    <full text of the proposed `tensions/<slug>.md`, per
    discover-design's template>
  - ...
  ```

  If a class is empty, write `(none found)`. Don't pad.
```

## What this skill does NOT do

- Does not modify any file under `.ok-planner/design/`. Findings
  are surfaced in a report; resolution is human-driven and rides
  through the spec pipeline.
- Does not auto-resolve semantic conflicts. Semantic disagreement
  goes into a follow-up `/refine-design` session as new tension
  entries (which get written by execute-plan, not by merge).
- Does not invoke `/brainstorm` or `/refine-design` automatically.
- Does not modify the merge commit. All changes are new working-tree
  edits (or new commits on top of HEAD, if the user chooses to commit
  them).
- Does not handle in-progress merge conflicts. Run `/merge` after a
  successful `git merge` or `git rebase`, not during conflict
  resolution.
