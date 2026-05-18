---
name: execute-plan
description: "ONLY activated by explicit /execute-plan slash command or ok-planner pipeline. Never auto-triggered."
---

# Executing a Plan to Completion (single subagent)

Dispatch one subagent with the full plan and trust it to finish. The plan is a record of the user's intent; the implementer's job is to deliver that intent. When the implementer reports DONE, run a divergence audit to capture creative choices, then hand off to `ok-planner:review-work`. The implementer is given two — and only two — permitted reasons to stop short. Reports that don't qualify as DONE or as a genuinely-valid BLOCKED are re-dispatched with a sharper directive; escalation to the user is reserved for real BLOCKED conditions and for the case where re-dispatch has run its cap.

The orchestrator dispatches one subagent for the whole plan. Goal: full completion in as few dispatches as possible, ideally one. Review happens once at the end. Reworks happen when the user asks for them — not because the agent decided to defer.

## Orchestrator's role

You are the taskmaster. You do not implement. You dispatch, verify the report, and hand off. You insist on full completion before review. You do not take BLOCKED at face value — agents under pressure misapply it, and your job is to push back when the stated reason doesn't actually match one of the two permitted bullets. Off-pattern reports get re-dispatched, not escalated.

## Process

1. **Read the plan and extract tasks.** Capture the full text of each task verbatim — you'll be passing them to the subagent. Number them as the plan numbers them. While reading, note the plan's `**Spec:**` header — you'll need it for the divergence audit and for archival.

2. **Initialize the layout.** Invoke `ok-planner:init` to ensure `.ok-planner/plans/` and `.ok-planner/history/` exist.

3. **Dispatch the implementer** using the subagent prompt template below.

4. **Parse the report and verify it.** Do not take the implementer's label at face value. The implementer is expected to ship the work; off-pattern reports usually mean the implementer wavered and should be nudged back, not that the user needs to intervene.

   - **STATUS: DONE** → confirm every task in the dispatch is marked done in the per-task list. If yes, proceed to step 5 (divergence audit). If DONE was claimed but some tasks are not-done, that's off-pattern — re-dispatch.
   - **STATUS: BLOCKED** → verify the stated reason genuinely matches one of the two bullets. Bullet 1 is credentials/access the user must provide; bullet 2 is an unauthorized destructive or irreversible action. Read the specific situation the implementer described and judge whether it actually fits. **Be skeptical.** Agents under pressure label coupling, ambiguity, "I'd rather not," "this needs discussion," and "the plan was wrong about X" as BLOCKED. None of those qualify. If the BLOCKED is genuine → escalate (the only first-line escalation path). If misapplied → re-dispatch with a directive that the reported reason is not a permitted BLOCKED and the implementer should figure it out from spec intent.
   - **Anything else** (STATUS: PARTIAL, STATUS: INCOMPLETE, no status, malformed report, zero meaningful changes to the working tree) → re-dispatch. Not an escalation.

   See "Re-dispatch" for the prompt shape and "Escalation" for the rare escalation cases.

5. **Divergence audit.** Dispatch the auditor (template below). It reads the plan, the spec, and the working-tree diff and produces `.ok-planner/plans/<plan-basename>-divergences.md`. Surface the report's path and a one-line summary to the user, then continue.

6. **Final review.** Invoke `ok-planner:review-work` for the full implementation. Let `review-cleanup` drive the fix cycle to clean.

6a. **Regenerate `concepts.md` if concepts/ was touched.** If any task in this plan added, modified, renamed, split, or merged a file under `.ok-planner/design/concepts/`, regenerate `.ok-planner/design/concepts.md` per the format documented in `ok-planner:discover-design`'s SKILL.md (sorted alphabetical bullet list of `<slug> — <one-sentence definition>`, optional `(aliases: ...)` parenthetical). Skip this step if `concepts/` was not touched or if `.ok-planner/design/concepts/` does not exist.

7. **Walk the divergence report with the user.** After review reports clean, surface the divergence entries. The user decides what (if anything) to act on — typically a rework if a divergence reveals a cleaner shape worth pursuing.

8. **Archive.** Move the plan, its divergence file, and the linked spec into
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

**`execute-plan` is the one skill that mutates the design docs.** The design docs are a source of truth with the same weight as code — they describe the project as it stands, and they change only through plan execution to keep code and design in sync. If the spec has a `## Design changes` section, `write-plan` will have turned each bullet into one or more plan tasks (mutate `concepts/<slug>.md`, move `tensions/<slug>.md` to `_resolved/`, create new concept files, etc.). The implementer carries out those tasks like any other; step 6a regenerates `concepts.md` (the TOC) afterward if any `concepts/` file was touched.

The implementer does not consult the design docs to decide what to build — that's already decided in the spec. The design docs as oracle are the concern of `review-holistic` (which uses them as the source of truth for whole-codebase convergence) and the read-only consultants (`brainstorm`, `refine-design`, `merge`) — those skills read the docs and capture any changes they need in a spec, which eventually flows back through this pipeline.

## Subagent prompt template

Dispatch a fresh `general-purpose` agent. Use this prompt verbatim, substituting the bracketed values:

Agent (general-purpose):
  You are the implementer for this plan. You're working in a fresh context — clean slate, plenty of room. The user designed this plan and trusts you to carry it out. The orchestrator is your only correspondent.

  ## Plan
  Path: [plan path]

  ## Spec (source of intent)
  Path: [spec path from the plan's **Spec:** header, or "none" if absent]

  The spec is the record of what the user wants — the business value, the reasoning, the intent behind every task. The plan is the route. When the plan's wording is imperfect or a detail doesn't quite fit reality, look at the spec for what the work is actually trying to accomplish and do that. The plan is a map, not a script.

  ## Tasks
  [Numbered list of tasks, full text verbatim from the plan — do not summarize, do not say "see the plan." The subagent should not need to look up tasks.]

  ## Trust and judgment

  The user has already weighed in by writing this plan. Every action it describes is authorized; you don't need to second-guess scope or ask for confirmation. They are not waiting for questions — they've handed off, and your job is to deliver the spec's business value.

  Use your judgment. Be creative. If you see a cleaner way to express something the plan describes, take it. If a task as written assumes a shape that doesn't quite match the code, look at the spec's intent and figure out what to build. Three similar lines is better than a premature abstraction, but a cleaner shape than the plan suggested is better than mechanical adherence. A separate divergence auditor will review the working tree after you finish and produce a record of creative choices for the user — your job is to ship the work, not to flag judgment calls back.

  Work patiently and thoroughly. There is no clock. Finishing well matters more than finishing fast.

  ## Reassurances
  - Context is the harness's concern, not yours — you have the room you need.
  - Build errors are part of the work; debug them one at a time, no rush.
  - Multi-file changes are normal — the plan accounts for the scope.
  - If a file or symbol the plan references doesn't quite exist or has a different shape than the plan assumes, that's information about the plan being slightly out of date — figure out what the spec is actually asking for and build it. Not a blocker.
  - If a task seems to conflict with something documented elsewhere (a concept, an invariant), spec intent wins for what to ship; the auditor surfaces the discrepancy for the user.
  - Anything authorized by the plan or implied by the spec's intent is safe to do.

  ## BLOCKED — the only two reasons to stop short

  1. **Credentials or access you cannot obtain.** You need a secret, an API key, a remote resource, or external access that only the user can provide, and there's no workaround.
  2. **An unauthorized destructive action.** A task would require something irreversible (delete branches, force-push, drop data, `rm -rf` outside the working tree) that the plan does not explicitly authorize.

  That's the full list. There is no third option. If you find yourself wanting to stop for any other reason — "this needs discussion," "the plan was wrong about X," "I want to surface a tradeoff," "I made a different choice and want approval," "this is more coupled than I expected" — that is not a blocker. Make the call from spec intent, implement, and the auditor will surface the divergence.

  ## How to report back

  When every task is done and its verification passes:

    STATUS: DONE

    Tasks:
    - 1: done — <one-line summary of what was built/changed>
    - 2: done — <one-line summary>
    - ...
    (One line per task. Every task gets a status.)

  If a permitted BLOCKED condition genuinely applies:

    STATUS: BLOCKED — bullet <1 or 2>

    <Specific situation: what credential is needed, or what destructive action the plan would require.>

    Tasks:
    - 1: done — <summary>
    - 2: not-done — blocked on bullet <1 or 2>
    - ...

  Do not invent other statuses. DONE means done; BLOCKED means one of the two bullets applies. If you're tempted to write "PARTIAL" or to list a task as not-done with any other reason, stop — the right answer is to deliver the work.

  ## Rules
  - Read files before editing.
  - Implement what each task is asking for. Deviations from the literal wording are fine when spec intent is clearer — the auditor will catch them.
  - Run the verifications each task specifies. If a verification fails, debug and fix — that's part of the task.
  - Do NOT commit. The user commits when they are ready.

## Divergence audit

When the implementer reports STATUS: DONE, dispatch a fresh `general-purpose` agent as the divergence auditor. This step is not a code review — it is a record of creative choices, produced for the user before review begins.

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

The orchestrator re-dispatches whenever the implementer's report is off-pattern — not a clean DONE and not a genuinely-valid BLOCKED. Re-dispatch is a sharpening, not a punishment: the same trust framing, the same template, with a short note from the orchestrator explaining why the prior report didn't qualify.

When you re-dispatch:
- Use the same subagent prompt template.
- Update the "Tasks" list to only include tasks not yet done.
- Insert a `## Note from the orchestrator` section above the tasks list with:
  - What the prior dispatch reported (one short paragraph, verbatim quotes where useful)
  - Specifically why it didn't qualify (e.g., "BLOCKED bullet 1 requires credentials only the user can provide; the situation you described — coupling between modules X and Y — does not match bullet 1. Use spec intent and deliver the work.")
  - A reminder of the trust frame: figure it out from spec intent, deliver the work, the auditor records divergences.
- Keep the positive framing of the rest of the prompt intact. The re-dispatch is firmer, not harsher.

**Cap.** After 2 re-dispatches (3 dispatches total) that still don't produce a clean DONE or a valid BLOCKED, escalate to the user. At that point the orchestrator's nudging isn't getting through and the user should decide.

## Escalation

Escalation is rare. Only two cases:

1. **Genuinely valid BLOCKED.** The implementer reported BLOCKED, and on inspection the reason really does match bullet 1 (credentials/access the user must provide) or bullet 2 (unauthorized destructive action). The implementer cannot proceed without user input. Surface the report verbatim, the plan path, and the remaining tasks. Wait for the user's decision.

2. **Re-dispatch cap reached.** After 3 dispatches the implementer still hasn't returned a clean DONE or a valid BLOCKED. Surface all dispatch reports verbatim, your read of why nudges aren't getting through, and the available options: (a) the user sharpens the brief themselves and you re-dispatch, (b) rework the plan, (c) abandon.

Anything else — PARTIAL, coupling discoveries, "I'd rather discuss this," misapplied BLOCKED, judgment-call requests — is not an escalation. Re-dispatch.

## Final review

After the divergence audit completes and you've surfaced it to the user, invoke `ok-planner:review-work` for the full implementation. Do NOT dispatch your own reviewer — `review-work` runs `review-cleanup`, which drives the fix cycle to clean.

After the review reports clean, walk the divergence entries with the user. Reworks come from this conversation — the user decides which (if any) divergences warrant going back and expressing the work more cleanly.

## Rules

- One subagent at a time. Never dispatch in parallel.
- The orchestrator dispatches and tracks progress; it does not implement.
- The orchestrator does not interrupt the subagent mid-dispatch.
- Review happens once, at the end — not between dispatches.
- Do not commit at any point. The user commits when ready.
- Never start work on `main`/`master` without explicit user consent.
