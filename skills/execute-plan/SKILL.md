---
name: execute-plan
description: "ONLY activated by explicit /execute-plan slash command or ok-planner pipeline. Never auto-triggered."
---

# Executing a Plan to Completion (pass-by-pass)

Dispatch one fresh subagent per pass. Between passes, gate on the pass's declared verification command (skip the gate for `broken-intentional` passes). When all passes are DONE, run a divergence audit to capture creative choices, then hand off to `ok-planner:review-work`.

The plan is a record of the user's intent; the implementer's job is to deliver the spec's business value pass by pass. The implementer is given two — and only two — permitted reasons to stop short within a pass. Reports that don't qualify as DONE on the dispatched pass or as a genuinely-valid BLOCKED are re-dispatched with a sharper directive; escalation to the user is reserved for real BLOCKED conditions and for the case where re-dispatch has run its cap on the current pass.

The orchestrator's job is to drive the whole plan to completion one pass at a time. Most large plans converge in 5–12 passes, so 5–12 subagent dispatches. Review happens once at the end. Reworks happen when the user asks for them — not because the agent decided to defer mid-plan.

## Orchestrator's role

You are the taskmaster. You do not implement. You dispatch each pass, verify the report, gate on the pass's verification command, and advance to the next pass. You insist on completing every pass before review. You do not take BLOCKED at face value — agents under pressure misapply it, and your job is to push back when the stated reason doesn't actually match one of the two permitted bullets. Off-pattern reports get re-dispatched on the current pass, not escalated.

## Process

1. **Read the plan and extract passes.** For each pass, capture: its number, name, **Goal**, **Scope** (task range), **End state** (`working` / `broken-intentional` with restorer named), **Verification** command, and the full verbatim text of every task it contains. You'll feed one pass at a time to the implementer. Note the plan's `**Spec:**` header — you'll need it for the per-pass dispatch, the divergence audit, and archival.

   If the plan has no passes (older flat-task plans), treat the entire task list as a single pass with `End state: working` and the plan's overall verification (if any). Flag this to the user as a one-line note before dispatching: the plan predates the pass model and the executor is treating it as a single pass.

2. **Affirm the layout.** Invoke `ok-planner:affirm` to ensure `.ok-planner/plans/` and `.ok-planner/history/` exist.

3. **Dispatch passes in order.** For each pass in plan order:

   a. **Dispatch the pass.** Dispatch a fresh `Agent` (general-purpose) using the pass-dispatch template below.

   b. **Parse the report and verify it.** Do not take the implementer's label at face value. The implementer is expected to ship the pass; off-pattern reports usually mean the implementer wavered and should be nudged back, not that the user needs to intervene.

      - **STATUS: DONE** → confirm every task listed in the dispatched pass is marked done in the per-task list. If yes, proceed to (c). If DONE was claimed but some tasks in this pass are not-done, that's off-pattern — re-dispatch the pass.
      - **STATUS: BLOCKED** → verify the stated reason genuinely matches one of the two bullets. Bullet 1 is credentials/access the user must provide; bullet 2 is an unauthorized destructive or irreversible action. Read the specific situation the implementer described and judge whether it actually fits. **Be skeptical.** Agents under pressure label coupling, ambiguity, "I'd rather not," "this needs discussion," and "the plan was wrong about X" as BLOCKED. None of those qualify. If the BLOCKED is genuine → escalate (the only first-line escalation path). If misapplied → re-dispatch the pass with a directive that the reported reason is not a permitted BLOCKED and the implementer should figure it out from spec intent.
      - **Anything else** (STATUS: PARTIAL, STATUS: INCOMPLETE, no status, malformed report, zero meaningful changes to the working tree) → re-dispatch the pass. Not an escalation.

   c. **Run the pass verification gate.** If the pass's **End state** is `working`, run its **Verification** command. If it exits 0, advance to the next pass. If it exits non-zero, re-dispatch the pass with the failed verification output included as a directive (see "Re-dispatch"). If the pass's **End state** is `broken-intentional`, skip the gate and advance — the broken state is the declared outcome and the next pass (or a later one) restores working state.

   d. **Sanity check before advancing.** If the just-completed pass was `broken-intentional` and the pass it named as the restorer doesn't exist in the plan (e.g., "restored by Pass 7" but the plan only has 6 passes), stop and escalate — the plan is internally inconsistent. This is a planner failure, not an implementer failure; the user needs to know.

   Continue until every pass has reported DONE and passed (or skipped) its verification gate.

4. **Divergence audit.** Dispatch the auditor (template below). It reads the plan, the spec, and the working-tree diff across the whole plan and produces `.ok-planner/plans/<plan-basename>-divergences.md`. Surface the report's path and a one-line summary to the user, then continue.

5. **Final review.** Invoke `ok-planner:review-work` for the full implementation. Let `review-cleanup` drive the fix cycle to clean.

5a. **Regenerate `concepts.md` if concepts/ was touched.** If any task across the whole plan added, modified, renamed, split, or merged a file under `.ok-planner/design/concepts/`, regenerate `.ok-planner/design/concepts.md` per the format documented in `ok-planner:discover-design`'s SKILL.md (sorted alphabetical bullet list of `<slug> — <one-sentence definition>`, optional `(aliases: ...)` parenthetical). Skip this step if `concepts/` was not touched or if `.ok-planner/design/concepts/` does not exist.

6. **Walk the divergence report with the user.** After review reports clean, present every divergence as a markdown table — not a summary, not "the key ones," not "here are the highlights." Every entry the auditor recorded goes in the table. The user decides what (if anything) to act on; that decision needs the full list, not your selection. Table format:

   | # | Pass / Task | What the plan said | What was implemented | Inferred reason |
   |---|-------------|--------------------|----------------------|-----------------|
   | 1 | Pass 2 / Task 5 | ... | ... | ... |

   Keep cells terse — one short phrase per cell, with a file:line where it helps. After the table, add one sentence inviting the user to flag any entry they want to act on. If the auditor reported "no meaningful divergences," say that in one line and skip the table — there's nothing to tabulate.

7. **Archive.** Move the plan, its divergence file, and the linked spec into
   `.ok-planner/history/`:
   - `<plan>.md` and `<plan>-divergences.md` → `.ok-planner/history/plans/`
   - spec from the plan's `**Spec:**` header →
     `.ok-planner/history/specs/`

   Use `git mv` if the project tracks `.ok-planner/` in git, otherwise
   plain `mv`. Skip the spec move if the `**Spec:**` field is `none`,
   missing, or points to a file that doesn't exist. Skip the divergence
   move if no divergence file was produced. Tell the user what was
   moved in one short line. This is the default end-of-run step;
   skip only if the user explicitly asked you not to archive.

## Source of truth and design docs

The spec is the source of truth for this skill and for `review-work`. Design considerations are factored into the spec during `brainstorm` (which reviews the spec against the design docs before it's approved); `write-plan` then translates that approved spec into tasks. By the time work reaches `execute-plan`, the spec is the authoritative record of intent. The implementer and the auditor look to the spec — not to the design docs under `.ok-planner/design/` — for the judgments they need to make.

**`execute-plan` is the one skill that mutates the design docs.** The design docs are a source of truth with the same weight as code — they describe the project as it stands, and they change only through plan execution to keep code and design in sync. If the spec has a `## Design changes` section, `write-plan` will have turned each bullet into one or more plan tasks (mutate `concepts/<slug>.md`, move `tensions/<slug>.md` to `_resolved/`, create new concept files, etc.). The implementer carries out those tasks like any other; step 5a regenerates `concepts.md` (the TOC) afterward if any `concepts/` file was touched.

The implementer does not consult the design docs to decide what to build — that's already decided in the spec. The design docs as oracle are the concern of `review-holistic` (which uses them as the source of truth for whole-codebase convergence) and the read-only consultants (`brainstorm`, `refine-design`, `merge`) — those skills read the docs and capture any changes they need in a spec, which eventually flows back through this pipeline.

## Pass-dispatch template

Use this for every pass on a fresh `general-purpose` agent. Substitute bracketed values.

Agent (general-purpose):
  You are the implementer for one pass of a multi-pass plan. You're working in a fresh context — clean slate, plenty of room. The user designed this plan and trusts you to carry out this pass. The orchestrator is your only correspondent and will dispatch the next pass to a fresh agent when this one is done.

  ## Plan
  Path: [plan path]

  ## Spec (source of intent)
  Path: [spec path from the plan's **Spec:** header, or "none" if absent]

  The spec is the record of what the user wants — the business value, the reasoning, the intent behind every task. The plan is the route. When the plan's wording is imperfect or a detail doesn't quite fit reality, look at the spec for what the work is actually trying to accomplish and do that. The plan is a map, not a script.

  ## This pass: [pass number and name, e.g., "Pass 2: Add session-based auth"]

  - **Goal:** [from the pass header]
  - **End state:** [working | broken-intentional (restored by Pass N)]
  - **Verification:** [command, or "none" for broken-intentional]
  - **Position in plan:** Pass [N] of [total]. [If not the last pass: "Subsequent passes handle [one-line summary of what's left]."] [If the last pass: "This is the final pass; the codebase must be working at the end."]

  Your job is to deliver this pass — not to peek ahead at other passes, not to do work that belongs to a later pass, not to leave this pass short. If a task in this pass implies cleanup that a later pass is explicitly responsible for, leave it for that pass.

  ## Tasks in this pass
  [Numbered list of tasks belonging to this pass, full text verbatim from the plan. Use the plan's global task numbers (e.g., Tasks 4, 5, 6 if this is Pass 2 of a plan with passes 1–3 covering Tasks 1–9). Do not summarize, do not say "see the plan."]

  ## End-state expectation

  [If `working`: "The codebase must verify cleanly at the end of this pass. After your final task, run the pass's Verification command and confirm it exits 0 before reporting DONE. If it fails, debug and fix — that's part of this pass."]

  [If `broken-intentional`: "This pass deliberately leaves the codebase non-working. Pass [N] is responsible for restoring it. Do NOT run the plan's overall build/test suite expecting it to pass — it won't, and that's intended. Do run any per-task verifications the pass specifies (e.g., `git status` confirming a file was deleted). Report DONE when every task in this pass is complete, regardless of whether the broader tree compiles."]

  ## Design-doc mutations (if any)

  If a task in this pass mutates a file under `.ok-planner/design/concepts/` or `.ok-planner/design/tensions/`, follow the concept self-containment rule from `ok-planner:discover-design`'s SKILL.md. In short: concept body has no file paths, no `code:`/`pkg:` citations, no external-doc references (`docs/...`, READMEs, sibling-repo paths), no quoted code or lint-config allowlists, no "Owns / Does NOT own" sections that name code paths. Allowed citations: other concept slugs, annotation IDs (`@blessed-invariant: N`), spec slugs in dated Notes entries, and dates. Tension `## Resolution candidates` sections are also path-free; `## What is muddy` / `## Evidence` can cite code as evidence.

  The spec's `## Design changes` section is the authoritative description of each mutation. If a bullet there leaks a path into the new concept body, apply the rule when implementing — rewrite the new text to be path-free. That counts as a divergence; the audit will surface it for the user.

  ## Trust and judgment

  The user has already weighed in by writing this plan. Every action it describes is authorized; you don't need to second-guess scope or ask for confirmation. They are not waiting for questions — they've handed off, and your job is to deliver this pass.

  Use your judgment. Be creative. If you see a cleaner way to express something the pass describes, take it. If a task as written assumes a shape that doesn't quite match the code, look at the spec's intent and figure out what to build. Three similar lines is better than a premature abstraction, but a cleaner shape than the plan suggested is better than mechanical adherence. A separate divergence auditor will review the working tree after the full plan finishes and produce a record of creative choices for the user — your job is to ship this pass, not to flag judgment calls back.

  Work patiently and thoroughly. There is no clock. Finishing well matters more than finishing fast.

  ## Reassurances
  - Context is the harness's concern, not yours — you have the room you need for this pass.
  - Build errors inside this pass are part of the work; debug them one at a time, no rush.
  - Multi-file changes are normal — the pass accounts for the scope.
  - If a file or symbol the plan references doesn't quite exist or has a different shape than the plan assumes, that's information about the plan being slightly out of date — figure out what the spec is actually asking for and build it. Not a blocker.
  - If a task seems to conflict with something documented elsewhere (a concept, an invariant), spec intent wins for what to ship; the auditor surfaces the discrepancy for the user.
  - Anything authorized by the plan or implied by the spec's intent is safe to do.

  ## BLOCKED — the only two reasons to stop short

  1. **Credentials or access you cannot obtain.** You need a secret, an API key, a remote resource, or external access that only the user can provide, and there's no workaround.
  2. **An unauthorized destructive action.** A task would require something irreversible (delete branches, force-push, drop data, `rm -rf` outside the working tree) that the plan does not explicitly authorize.

  That's the full list. There is no third option. If you find yourself wanting to stop for any other reason — "this needs discussion," "the plan was wrong about X," "I want to surface a tradeoff," "I made a different choice and want approval," "this is more coupled than I expected" — that is not a blocker. Make the call from spec intent, implement, and the auditor will surface the divergence.

  ## How to report back

  When every task in this pass is done (and, for `working` passes, the Verification command exits 0):

    STATUS: DONE

    Pass: [pass number and name]
    Tasks:
    - [N]: done — <one-line summary of what was built/changed>
    - [N+1]: done — <one-line summary>
    - ...
    Verification: [pass | skipped (broken-intentional)]

  If a permitted BLOCKED condition genuinely applies:

    STATUS: BLOCKED — bullet <1 or 2>

    Pass: [pass number and name]
    <Specific situation: what credential is needed, or what destructive action the plan would require.>

    Tasks:
    - [N]: done — <summary>
    - [N+1]: not-done — blocked on bullet <1 or 2>
    - ...

  Do not invent other statuses. DONE means this pass is done; BLOCKED means one of the two bullets applies. If you're tempted to write "PARTIAL" or to list a task as not-done with any other reason, stop — the right answer is to deliver this pass.

  ## Rules
  - Read files before editing.
  - Implement what each task is asking for. Deviations from the literal wording are fine when spec intent is clearer — the auditor will catch them.
  - Run the verifications each task specifies. For `working` passes, also run the pass-level Verification command before reporting DONE.
  - Stay within this pass. Don't do work belonging to other passes.
  - **Checkpoint after every task.** When a task is complete, run `git add -A` to stage your progress into the index. This is a checkpoint, not a commit — it needs no message, costs nothing, and moves the work into the index where a stray working-tree revert can't reach it. The index is the only safety net during a run (nobody commits between passes), so checkpoint diligently — every prior task and every prior pass left uncommitted work in this tree.
  - **Never revert, reset, or clean the tree.** Do not run `git checkout -- <path>` / `git checkout .`, `git restore`, `git reset` (any mode), `git stash`, or `git clean` — not even on a single file. A file you think you "only" touched just now may carry edits from earlier tasks or passes, and a revert takes those with it; there is no commit to recover them from. When an edit goes wrong, **fix it forward** — edit the file again until it says what you want. The reflexive `git checkout -- .` to "start over" is the exact footgun this rule exists to prevent: it erases everything not yet committed.
  - Do NOT commit. Staging (`git add`) is your checkpoint; committing is the user's call, made after the run.

## Re-dispatch directive template (verification failed on a `working` pass)

When step 3(c) finds a `working` pass's Verification exited non-zero, re-dispatch the same pass to a fresh implementer with this directive prepended above the pass-dispatch template:

  Pass [N] reported DONE but its Verification command failed:

      $ [command]
      [first ~40 lines of output, or full output if shorter]

  Debug and fix. The pass isn't done until Verification passes. Report DONE again when it does, or BLOCKED only if a permitted bullet applies.

## Divergence audit

After every pass has reported DONE and passed (or skipped) its verification gate, dispatch a fresh `general-purpose` agent as the divergence auditor. The auditor needs a cold reader's view of the diff, uncolored by what the implementer thought it was doing. This step is not a code review — it is a record of creative choices, produced for the user before review begins.

Agent (general-purpose):
  You are the divergence auditor for a plan the implementer just finished. Your only job is to produce an honest, useful record of where the working tree differs from what the plan literally said, so the user knows what creative choices were made.

  ## Read
  - Plan: [plan path]
  - Spec: [spec path, or "none" if the plan's **Spec:** is none/absent]
  - Working-tree diff: run `git diff`, `git diff --staged`, and `git status` for untracked files. If there are submodule changes, `git diff --submodule=diff`.
  - Modified and new files in full, where the diff alone doesn't give enough context.

  ## Produce
  A markdown report at [divergence path]. For each meaningful place where the implementation differs from what the plan literally said, record:

  - **What the plan said:** the relevant excerpt or task number
  - **What was implemented:** what actually landed (with file:line where useful)
  - **Inferred reason:** your best read — plan error, cleaner shape, forced choice, spec intent override, etc.

  Meaningful means: a choice a thoughtful reviewer would want to know about. Skip stylistic differences, trivial naming, and obvious equivalents. Focus on:
  - Different file structure than the plan named
  - Different function/type/module shape than the plan described
  - Extra helpers or files the plan didn't mention
  - Omitted plan steps the implementer judged unnecessary
  - Substituted approaches (different library, different algorithm, different data shape)
  - Concept or invariant tensions the implementer resolved by reading spec intent

  If the implementation matches the plan throughout, write a one-line "no meaningful divergences" report and finish. Honest "no divergences" is a valid outcome.

  Do not propose fixes — this is a record, not a critique. A separate review step (`review-work`) handles correctness.

  Save the report to [divergence path] and reply with:

    STATUS: DONE
    Divergences recorded: <count, or "none">

  The count helps the orchestrator give the user a one-line summary without re-reading the report.

Save path: `.ok-planner/plans/<plan-basename>-divergences.md`. After the auditor returns DONE, surface the report path and a one-line summary (e.g., "auditor recorded 3 divergences" or "no meaningful divergences") to the user, then proceed to `ok-planner:review-work`.

## Re-dispatch

The orchestrator re-dispatches whenever the implementer's report on the current pass is off-pattern — not a clean DONE on this pass and not a genuinely-valid BLOCKED. Re-dispatch is a sharpening, not a punishment: same trust framing, same template, with a short note from the orchestrator explaining why the prior report didn't qualify.

When you re-dispatch a pass:
- Use the pass-dispatch template on a fresh `Agent`.
- Update the "Tasks in this pass" list to only include tasks within this pass that aren't yet done.
- Insert a `## Note from the orchestrator` section above the tasks list with:
  - What the prior dispatch reported (one short paragraph, verbatim quotes where useful)
  - Specifically why it didn't qualify (e.g., "BLOCKED bullet 1 requires credentials only the user can provide; the situation you described — coupling between modules X and Y — does not match bullet 1. Use spec intent and deliver the pass." Or: "Verification command exited 2; debug and fix.")
  - A reminder of the trust frame: figure it out from spec intent, deliver the pass, the auditor records divergences.
- Keep the positive framing of the rest of the prompt intact. The re-dispatch is firmer, not harsher.

**Cap (per pass).** After 2 re-dispatches on the same pass (3 dispatches total for that pass) that still don't produce a clean DONE or a valid BLOCKED, escalate to the user. The cap is per-pass, not per-plan — earlier passes that succeeded don't count.

## Escalation

Escalation is rare. Only three cases:

1. **Genuinely valid BLOCKED.** The implementer reported BLOCKED on the current pass, and on inspection the reason really does match bullet 1 (credentials/access the user must provide) or bullet 2 (unauthorized destructive action). The implementer cannot proceed without user input. Surface the report verbatim, the plan path, the current pass, and the remaining passes. Wait for the user's decision.

2. **Re-dispatch cap reached on a pass.** After 3 dispatches on the same pass, the implementer still hasn't returned a clean DONE or a valid BLOCKED. Surface all dispatch reports for that pass verbatim, your read of why nudges aren't getting through, and the available options: (a) the user sharpens the brief themselves and you re-dispatch, (b) rework the pass (or the plan), (c) abandon.

3. **Plan-level inconsistency.** Step 3(d) found a `broken-intentional` pass naming a restorer that doesn't exist in the plan. The plan is internally inconsistent and `execute-plan` cannot proceed safely. Surface the offending pass and the missing restorer; the user takes it back to `write-plan` (or fixes the plan inline).

Anything else — PARTIAL, coupling discoveries, "I'd rather discuss this," misapplied BLOCKED, judgment-call requests — is not an escalation. Re-dispatch.

## Final review

After the divergence audit completes and you've surfaced it to the user, invoke `ok-planner:review-work` for the full implementation. Do NOT dispatch your own reviewer — `review-work` runs `review-cleanup`, which drives the fix cycle to clean.

After the review reports clean, walk the divergence entries with the user using the table format from Process step 6 (every entry, not a selection). Reworks come from this conversation — the user decides which (if any) divergences warrant going back and expressing the work more cleanly. They can only make that call against the full list.

## Rules

- One subagent at a time. Never dispatch passes in parallel.
- The orchestrator dispatches each pass, gates on its verification, and advances; it does not implement.
- The orchestrator does not interrupt the subagent mid-dispatch.
- Review happens once, after all passes are DONE — not between passes.
- Per-pass verification (between passes) is not "review" — it's a fast machine check that exits 0 or doesn't. If you need human judgment about whether a pass succeeded, the verification command isn't tight enough; that's a plan defect, not a step to insert here.
- Do not commit at any point. The user commits when ready.
- Never revert, reset, or clean the working tree — this binds the orchestrator as well as every dispatched agent. No `git checkout -- <path>`, `git restore`, `git reset`, `git stash`, or `git clean`, even on a single file. Uncommitted work from earlier passes lives all over the tree with no commit to recover it; a revert destroys it silently. Implementers checkpoint each task with `git add -A` (staging, not committing) so the index always holds known-good progress; recovery from any botched edit is fix-forward, never undo. If you need to roll a pass back to a clean baseline, that is a plan/escalation matter for the user — not a `git reset` you run yourself.
- Work proceeds on whatever branch is currently checked out, including `main`/`master`. Do not create branches, switch branches, or stop to ask which branch to use — branch/worktree setup is the caller's concern (see `execute-plan-in-worktree` for the isolated-branch variant).
