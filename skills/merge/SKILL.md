---
name: merge
description: "ONLY activated by explicit /merge slash command. Never auto-triggered by conversation content."
---

# Merge: Triage a Merge or Rebase

Two-stage review of a recently-merged-in change set. Stage 1 triages the
design log: mechanical issues get fixed, drift gets recorded as Notes
appends, semantic disagreement becomes new tension entries. Stage 2 runs
a code review of the merge diff via `ok-planner:review-commits`, which
consults the (now-triaged) design log to inform its findings.

If the project has no design log (`.ok-planner/design/concepts/` does
not exist), stage 1 is skipped silently and `/merge` is effectively a
wrapper around `/review-commits` over the merge range.

## Why this exists

A merge or rebase can introduce two kinds of inconsistency:

- **Design log inconsistencies** — two branches edited the same concept
  incompatibly, or code that just landed contradicts a concept's
  documented Boundaries / Invariants. The design log is the project's
  oracle for "what was intended"; it must be coherent before a reviewer
  can use it. `/merge` doesn't *resolve* design ambiguity at merge time
  — it routes each kind of inconsistency to the appropriate shape:
  mechanical fix, Notes append, or new tension entry.
- **Code issues** — the usual review surface, applied to the merge diff.

`/merge` runs both stages, design log first, because stage 2 consults
the design log.

## When to invoke

Right after a git merge or rebase that touched code. Optionally before
a merge as a dry-run by checking out the merge base and running with the
prospective tip as the comparison ref.

## Inputs

- `.ok-planner/design/` — the current design log (post-merge), if present.
- The merge diff: `git diff <base>..HEAD`, where `<base>` defaults to
  `HEAD^1` if HEAD is a merge commit, else `main`. Override with the
  skill's optional argument: `/merge main`, `/merge HEAD~5`, etc.

## Process

1. **Resolve the base ref.** If HEAD is a merge commit and no argument
   given, use `HEAD^1`. Otherwise default to `main`. Allow the user to
   override.

2. **Stage 1 — Design log triage.** If `.ok-planner/design/concepts/`
   does not exist, skip this stage silently — the design log is opt-in.
   Otherwise:

   a. Dispatch the design-log triage subagent (prompt below) to surface
      findings in three classes:
      - **Mechanical** — slug collisions, malformed templates, broken
        `@concept:` annotations.
      - **Drift** — a concept's Boundaries / Invariants now contradicts
        code that just landed; or `@concept:` annotations point at
        sites that have moved or no longer enforce what the concept
        claims.
      - **Semantic conflicts** — two branches edited the same concept's
        Definition / Purpose / Boundaries / Invariants incompatibly,
        or added concept files for arguably the same noun under
        different slugs.

   b. Walk each finding with the user:
      - Mechanical: confirm proposed fix; apply in a batch once
        confirmed.
      - Drift: confirm the proposed Notes append; apply.
      - Semantic conflicts: confirm the drafted `tensions/<slug>.md`
        entry text; write the tension file. Do NOT try to resolve the
        conflict here — capturing it as a tension *is* the resolution
        at merge time.

   c. Collect the slugs of any new tensions written; carry them to
      the end-of-run summary.

   Single pass — no fix loop. The user is the arbiter; once each
   finding has a confirmed resolution, stage 1 is done.

3. **Stage 2 — Code review.** Invoke `ok-planner:review-commits` with
   the range `<base>..HEAD` as its skill argument so its interactive
   range-picker is bypassed. `/review-commits` already consults the
   design log when present (now triaged, per stage 1) and pipes its
   findings through `review-cleanup`. Let it run to completion.

4. **End-of-run summary.** Report what stage 1 did (counts by class)
   and what stage 2 did (findings + cleanup status). If stage 1 wrote
   any new tension entries, list their slugs and suggest
   `/refine-design` when the user is ready to resolve them. Do not
   invoke `/refine-design` automatically — it's a substantial
   interactive session and may not be the right moment.

## Design-Log Triage Subagent Prompt

```
Agent (general-purpose):
  ## Triage Design Log Against Merge

  ### Your job

  A merge or rebase has just landed. The design log at
  `.ok-planner/design/` may now be inconsistent with itself, with the
  codebase, or both. Surface the inconsistencies in three classes of
  finding. Do NOT attempt to resolve them — your job is to draft
  proposed actions; the orchestrator walks them with the user.

  ### Inputs

  - Current design log: read every file under
    `.ok-planner/design/concepts/` and `.ok-planner/design/tensions/`.
    Read `.ok-planner/design/concepts.md` (the TOC) for orientation.
  - Merge diff: `git diff <BASE>..HEAD` where BASE was supplied as
    `<BASE_REF>`. Read the diff in full; identify what code changed.
  - Optionally: `git log <BASE>..HEAD --oneline` for commit-level
    structure.

  Do NOT read existing docs (READMEs, narrative under `docs/`, etc.).
  The design log is the authoritative record of intent for this
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

  For each drift finding, draft a proposed Notes append for the
  affected concept file:
  - Concept file path
  - Specific evidence file:line where the drift shows up
  - One-line description of the observation
  - Proposed Notes append text (one or two lines, dated, citing the
    merge)

  Per the design log's format-durability rules, do NOT propose
  rewrites of Definition / Purpose / Boundaries / Invariants. Notes
  appends are the only safe mutation at merge time; rewrites are
  `/refine-design`'s job.

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

- Does not auto-resolve semantic conflicts. Semantic disagreement
  becomes a tension entry; resolving the tension is `/refine-design`'s
  job.
- Does not rewrite Definition / Purpose / Boundaries / Invariants in
  concept files. Only Notes appends are allowed at merge time.
- Does not invoke `/refine-design` automatically.
- Does not modify the merge commit. All changes are new working-tree
  edits (or new commits on top of HEAD, if the user chooses to commit
  them).
- Does not handle in-progress merge conflicts. Run `/merge` after a
  successful `git merge` or `git rebase`, not during conflict
  resolution.
