---
name: execute-plan
description: "ONLY activated by explicit /execute-plan slash command or ok-planner pipeline. Never auto-triggered."
---

# Executing a Plan to Completion (single subagent)

Dispatch one subagent with the full plan. Trust it to finish. If it returns with work undone but made progress, re-dispatch with only the remaining tasks. If it returns with zero progress, escalate to the user. After completion, hand off to `ok-planner:review-work`.

The orchestrator dispatches one subagent for the whole plan and re-dispatches only if the subagent returned without finishing. Goal: full completion in as few dispatches as possible, ideally one. Review happens once at the end.

## Orchestrator's role

You are the taskmaster. You do not implement. You dispatch, parse the subagent's report, decide what to do next, and re-dispatch if needed. You insist on full completion before review.

## Process

1. **Read the plan and extract tasks.** Capture the full text of each task verbatim — you'll be passing them to the subagent. Number them as the plan numbers them.

2. **Initialize the layout and the implementation notes file.** Invoke `ok-planner:init` first to ensure `.ok-planner/plans/` and `.ok-planner/history/` exist. Then create the notes file at the plan's filename with `.md` replaced by `-notes.md` (`.ok-planner/plans/<plan>.md` → `.ok-planner/plans/<plan>-notes.md`) with a header. The notes file is shared across all dispatches. While you're reading the plan, also note its `**Spec:**` header — you'll need it for archival at the end.

3. **Dispatch the subagent** using the prompt template below. On the first dispatch, "tasks remaining" is the full list.

4. **Parse the subagent's report.** It returns a structured per-task status. Compute:
   - Tasks done before this dispatch (from your tracking)
   - Tasks done after this dispatch (from this report)
   - Tasks newly completed (the delta)
   - Tasks still remaining

5. **Decide the next move:**
   - **All tasks done** → step 6.
   - **STATUS: BLOCKED** → engage the user. Surface the subagent's BLOCKED report verbatim, the notes file path, and which tasks remain. Do not re-dispatch.
   - **PARTIAL with zero new completions this dispatch** → escalate to the user (see "Escalation" below). Do not re-dispatch.
   - **PARTIAL with at least one new completion** → re-dispatch with the updated "tasks remaining" list. There is no cap on dispatches as long as each one makes net progress.

6. **Final review.** Once all tasks are done, invoke `ok-planner:review-work` for the full implementation. Let `review-cleanup` drive the fix cycle to clean.

6a. **Regenerate `concepts.md` if concepts/ was touched.** If any task in this plan added, modified, renamed, split, or merged a file under `.ok-planner/design/concepts/`, regenerate `.ok-planner/design/concepts.md` per the format documented in `ok-planner:discover-design`'s SKILL.md (sorted alphabetical bullet list of `<slug> — <one-sentence definition>`, optional `(aliases: ...)` parenthetical). Skip this step if `concepts/` was not touched or if `.ok-planner/design/concepts/` does not exist.

7. **Walk the notes file with the user.** After review reports clean, surface the notes-file entries. The user decides what to act on.

8. **Archive.** Move the plan, its notes file, and the linked spec into
   `.ok-planner/history/`:
   - `<plan>.md` and `<plan>-notes.md` → `.ok-planner/history/plans/`
   - spec from the plan's `**Spec:**` header →
     `.ok-planner/history/specs/`

   Use `git mv` if the project tracks `.ok-planner/` in git, otherwise
   plain `mv`. Skip the spec move if the `**Spec:**` field is `none`,
   missing, or points to a file that doesn't exist. Tell the user what
   was moved in one short line. This is the default end-of-run step;
   skip only if the user explicitly asked you not to archive.

## Design log awareness

If `.ok-planner/design/concepts/` exists, it is the project's
canonical concept catalog. The plan was written assuming the catalog
is correct; the subagent treats it as the design oracle while
implementing. The subagent's prompt below includes a "Design log"
section that hands the catalog to the implementer; the orchestrator
does not need to do anything special here.

If `.ok-planner/design/concepts/` doesn't exist, the subagent prompt's
"Design log" section is a no-op. Do not create the directory as a
side effect.

## Subagent prompt template

Dispatch a fresh `general-purpose` agent on each round. Use this prompt verbatim, substituting the bracketed values:

Agent (general-purpose):
  You are an implementer subagent dispatched by the ok-planner:execute-plan orchestrator. You are working in a fresh context — clean slate, plenty of room. The orchestrator is your only correspondent.

  ## Plan
  Path: [plan path]

  ## Notes file
  Path: [notes-file path]

  The notes file is the durable record of deviations, judgment calls, and items for post-run discussion. If prior subagents have worked on this plan, their entries are there. Read it first.

  ## Tasks remaining for this dispatch
  [Numbered list of tasks not yet complete. Paste each task's full text verbatim from the plan — do not summarize, do not say "see the plan." The subagent should not need to look up tasks.]

  ## How this works
  The user designed this plan and dispatched you to carry it out. The plan is your authorization — every action it describes is in scope. You don't need to second-guess scope or ask for re-confirmation on individual tasks. Trust the work that went into it.

  The user has handed off and is not waiting on questions. They've trusted you with the work. Focus on the plan, not on what to ask.

  Work patiently and thoroughly. There is no clock. Take the time you need to do the work properly — finishing well matters more than finishing fast.

  You have everything you need. If you genuinely don't, the BLOCKED list below covers what to do.

  ## Reassurances
  - Context is the harness's concern, not yours — you have the room you need.
  - Build errors are part of the work; debug them one at a time, no rush.
  - Multi-file changes are normal — the plan accounts for the scope.
  - Anything authorized by the plan is safe to do; the user has already weighed in by writing it.

  ## BLOCKED — the only reasons to stop short
  - You need credentials, access, or a resource you do not have and cannot obtain.
  - A task depends on a file or module the plan assumes exists but doesn't, and the plan does not tell you how to create it.
  - A step would perform a destructive or irreversible action (delete branches, force-push, drop data, `rm -rf` outside the working tree) that the plan does not authorize.

  If your reason to stop isn't on that list, log it in the notes file and keep going. That's what the notes file is for. Trust your judgment.

  ## Notes file entries
  As you work, append entries for anything the user would want to know after the run — deviations, judgment calls, discoveries, concerns. Format established by other ok-planner skills: a `## Task N — <title>` header followed by `**Deviation:**`, `**Reason:**`, and `**Surfaced for:**` fields.

  ## How to report back
  When you are done (or genuinely BLOCKED), reply with:

    STATUS: <DONE | PARTIAL | BLOCKED>

    Tasks:
    - 1: done — <one-line summary of what was done>
    - 2: not-done — <reason: still in progress, etc.>
    - ...
    (One line per task in the "tasks remaining" list above. Every task gets a status.)

    Notes-file entries appended this dispatch: <count, or "none">

    If BLOCKED: which bullet from the BLOCKED list applies, and the specific situation.

  STATUS: DONE means every task in the "tasks remaining" list above is now done. PARTIAL means you've done some but not all, with no BLOCKED condition met. BLOCKED means you've hit one of the listed BLOCKED conditions.

  ## Design log
  If `.ok-planner/design/concepts/` exists, it is the project's
  canonical concept catalog. To find what applies to a file you're
  about to edit:

  1. Read `.ok-planner/design/concepts.md` (the auto-generated TOC,
     small, always-readable). It lists every concept with a
     one-sentence definition.
  2. Grep for `@concept:` annotations in scope files —
     `rg '@concept:' <path>`. Each annotation names a concept slug
     that is load-bearing at that site.
  3. Read `concepts/<slug>.md` in full for any concept surfaced by
     (1) or (2) that bears on your task.

  Code you write must respect each consulted concept's stated
  boundaries and invariants. If a task would force a violation and
  the spec's `## Tensions resolved` section (if any) doesn't
  authorize it, that's a BLOCKED-class situation — log it in the
  notes file with `**Surfaced for:** user`, stop the task, and
  report PARTIAL. Do not silently break invariants.

  **Annotate as you work.** If you consulted a concept to understand
  or modify a file, leave a `@concept: <slug>` annotation at the
  most-specific load-bearing site (type definition, function head,
  call site — same granularity discipline as `@blessed-invariant`).
  This makes the next agent's lookup cheap. Do not carpet-bomb
  every file that happens to touch a concept; only annotate where
  the concept is enforced or expressed.

  If `.ok-planner/design/concepts/` doesn't exist, ignore this
  section.

  ## Rules
  - Read files before editing.
  - Implement what each task specifies; deviations go in the notes file, not in the behavior.
  - Run the verifications each task specifies. If a verification fails, debug and fix — that's part of the task.
  - Do NOT commit. The user commits when they are ready.

## Re-dispatch

When re-dispatching after a PARTIAL with progress:
- Use the same prompt template.
- Update "tasks remaining" to omit tasks marked done in any prior dispatch.
- The new subagent will read the notes file to pick up context from prior dispatches.
- Use the same positive framing. Do not add pressure to the re-dispatch prompt — the work continues from where it stood.

## Escalation (zero progress)

When a dispatch makes zero net progress, do NOT silently re-dispatch. Engage the user with:

- Plan path and notes file path
- Each prior subagent's status report, verbatim
- The tasks still remaining
- Your read of why progress stalled (e.g., "subagent reported BLOCKED on task 4 with no matching BLOCKED bullet," or "subagent reported DONE on tasks 1-3 but did not address tasks 4-6 in either dispatch")

Then wait for the user's decision — continue, modify the plan, abandon. Do not act unilaterally.

## Final review

Once all tasks are complete, invoke `ok-planner:review-work` for the full implementation. Do NOT dispatch your own reviewer — `review-work` runs `review-cleanup`, which drives the fix cycle to clean.

After the review reports clean, walk the notes-file entries with the user.

## Rules

- One subagent at a time. Never dispatch in parallel.
- The orchestrator dispatches and tracks progress; it does not implement.
- The orchestrator does not interrupt the subagent mid-dispatch.
- Review happens once, at the end — not between dispatches.
- Do not commit at any point. The user commits when ready.
- Never start work on `main`/`master` without explicit user consent.
