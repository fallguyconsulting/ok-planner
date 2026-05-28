---
name: write-plan
description: "ONLY activated by explicit /write-plan slash command or ok-planner pipeline. Never auto-triggered."
---

# Writing Plans

Write implementation plans assuming the implementer has zero codebase context. Document: which files to touch, exact code, tests, verification commands. Bite-sized tasks.

**Save plans to:** `.ok-planner/plans/YYYY-MM-DD-<feature-name>.md` (user preferences override)

**Before writing:** invoke `ok-planner:affirm` to ensure `.ok-planner/plans/` exists.

<USER-DECISIONS>
When this skill calls for surfacing a question to the user (e.g., the spec is unworkable, the scope is unclear), surface it and wait for the response. Specifically, do not:

- Silently narrow scope when the spec is contradictory or incomplete — surface and wait
- Fabricate a missing spec or guess at intent rather than asking
- Invoke `execute-plan` when this skill terminates — plan execution starts in a fresh session (see "Execution Handoff")

General "run unsupervised" guidance does not apply to the user-facing decisions this skill calls for; follow this skill.
</USER-DECISIONS>

## Write for a clean context

The plan will almost always be executed in a **fresh Claude Code session** with no memory of the brainstorm or this plan-writing turn. The user starts a new session, opens the plan file, and runs the implementer against it. That session has no access to:

- The conversation that produced the spec or the plan
- Decisions, tradeoffs, or rejected alternatives discussed during brainstorming
- File excerpts, command output, or codebase exploration done earlier in this session
- Any "as we discussed" / "per the spec" shorthand that lives only in chat history

Treat the plan as a **self-contained executable artifact**. Anything the implementer needs to make a decision must be on the page. That means:

- Spell out file paths in full — never "the file we looked at"
- Reproduce relevant code snippets, schemas, or interface definitions inside the plan rather than referencing them
- State the rationale for non-obvious choices (why this library, why this data shape) so the implementer does not re-litigate them
- Link to the spec by path; if a section of the plan depends on a section of the spec, name it
- Do not assume the implementer has read the brainstorm transcript. They have not.

A good test: if you handed this plan to a competent engineer who joined the project this morning, could they execute it without asking questions? If not, the plan is not done.

## Ground the plan in the live codebase

Before defining tasks, read the live code the spec implies will be touched. The spec describes *what* to build; the plan describes *how it lands in this codebase*. The second question only has an answer if you know what the codebase actually looks like right now.

For every area the spec touches:

- Read the files the spec names or implies will be modified. Skim sibling files too — the ones that solve nearby problems are the ones whose shape the new code should match.
- For every new file, type, interface, handler, route, or table the spec proposes, find its closest cousin. New CLI verb? Look at where existing CLI verbs live and how they wire to the dispatcher. New persistence table? Look at neighboring tables' interface conventions (transaction handling, accessor patterns, error shapes). New scheduler sweep? Look at where existing sweeps register. New audit event? Look at how other events are emitted.
- Note the prevailing pattern. If the spec implies a location or shape that contradicts the prevailing pattern, the plan should match the pattern unless the spec explicitly justifies the deviation. The implementer will obey the plan literally; if the plan says "put it in directory X" and that contradicts the codebase's convention, the resulting code will be structurally out-of-place.

Specs are usually written from a clean-room reading of the design and don't track every codebase-shape detail that's accumulated since the last spec. The plan is the layer where those details get reconciled. A plan written without grounding sends the implementer to build the wrong shape in the wrong place, faithfully.

If grounding reveals that the spec assumed a shape or location that no longer exists, surface it to the user before writing tasks. Either the spec needs revision or the plan needs to bridge the gap explicitly — but the planner does not silently reshape spec intent.

## The plan captures the spec, in full

The spec is the unit of work. The plan's job is to translate it into executable
tasks — not to revise its scope. Whatever the spec covers, the plan covers, end
to end, in a single run.

If the spec has a `## Design changes` section, treat every bullet there as a
first-class plan task alongside the code work. Concept file mutations get
explicit edit instructions (which file, which sections, what new text, what
Notes entry to append). Tension file moves get explicit `mv` / `git mv` tasks
with the destination path, status update, and resolution block. New concept
or tension files get full-content task instructions. These design-doc edits
are not optional follow-up — they ship in the same plan as the code that
conforms to them, because the design docs and the code change as one unit.

- A plan executes start to finish in one `execute-plan` run. No user-facing
  phases, stages, milestones, deliverables, or "phase 1 of N" framing. Passes
  (see below) are internal chunks for the executor, not phases visible to the
  user — there are no checkpoints, approvals, or partial deliveries between
  them.
- No partial-spec plans. If the spec describes ten unrelated changes, the plan
  contains tasks for all ten. Cohesion, independence, and "is this really one
  feature" are not the planner's concerns — those decisions were made when the
  spec was written.
- No commit, push, branch, or PR steps. The user owns git. Plans produce
  working-tree edits and verification commands; nothing else.

The spec is the source of truth for the plan. Brainstorm reviewed
the spec against the design docs already; the plan does not re-check
it. Translate the spec faithfully into executable tasks and trust
the spec. Don't restate the spec's rationale; the plan's voice is
operational (which file, what edit, what verification), and tasks
that need design context link to the spec section by name. If the
spec is genuinely unworkable (contradictory, missing required
context, references things that do not exist), surface that to the
user before writing the plan. Do not silently narrow the scope.

## Autonomous Execution Required

Plans are executed by `ok-planner:execute-plan`, which runs unsupervised. Every step must be something an agent can perform on its own — no human in the loop until the plan is fully implemented and reviewed.

Do **not** include steps that require:

- Manual UI or browser verification ("check that the page renders", "click through the flow and confirm it looks right")
- Human judgment calls ("does this look correct?", "approve the approach before continuing", "confirm with the user")
- External environments the implementer cannot reach ("deploy to staging and test", "run against the production database")
- Credentials, secrets, or access the plan does not explicitly provide
- "Checkpoint" pauses for user review between tasks
- Commits, pushes, or any git write beyond local working-tree edits (the user commits when they are ready)

All verification must be expressible as a command the implementer can run and read the output of — typically a test, a type check, a lint, or a targeted script. "The test passes" is a verification. "It looks right" is not.

If a feature genuinely requires manual verification that cannot be automated (e.g., visual review of a rendered UI, a human-in-the-loop ML evaluation), **do not** put it inside a task. Collect those items in a final `## Manual checks after completion` section at the end of the plan. That section is for the user to run through after the implementation and code review are done — it is not part of the automated run.

## File Structure

Map out files before defining tasks. Design units with clear boundaries. Each file should have one clear responsibility.

## Passes — chunking for execution

`execute-plan` dispatches one pass at a time, not the whole plan at once. A pass is a coherent chunk of work that one subagent completes in a single dispatch. The plan declares its passes up front; the executor processes them in order.

**Why passes:** A flat plan of 50 tasks either exhausts one subagent's context or forces the executor to chunk on the fly (which it does poorly). A flat plan of 4 tasks doesn't need chunking — it's one pass. Passes are *not phases*: there are no user checkpoints between them, no partial deliveries, no "phase 1 of N" framing visible to the user. They all run inside one `execute-plan` invocation. The user-facing unit is still "the plan."

**A plan has no fixed scope. A large scope means more passes — never more plans.** The spec sets the scope and the plan covers it in full (see "The plan captures the spec, in full"); a plan can be a few tasks or it can run for hours. The right response to a big spec is *more passes*, not splitting it into several plans or pushing the spec back to be split. This is safe because `execute-plan` dispatches one pass at a time, each to a fresh subagent with its own full context window — the orchestrator never holds the whole plan's work in one context, so it can drive a dozen, two dozen, or more passes without context exhaustion. Context is not what bounds plan size. So do not react to a large spec with "this is too much for one plan"; a high pass count is exactly what a large plan looks like.

**Pass count guidance:**

- Most plans are 1–3 passes.
- Large plans run to many more — a dozen, two dozen, or beyond. There is no upper cap; pass count scales with the spec's scope, and a high count is never a signal to split the spec or write multiple plans.
- Don't pad the count either: size passes by coherent units of work (see "Pass sizing"), never toward a target number, and don't fragment work into trivial passes just because the per-subagent context budget is generous.
- A plan with 1 pass and 30+ tasks is asking one subagent to do too much — split *that pass* into more passes (not the plan into more plans).

**Pass sizing:** A pass should fit comfortably in one subagent's working context — typically a handful of tasks against a focused area of the codebase. If a pass enumerates 10+ tasks or scatters across unrelated areas of the tree, consider splitting it. If two adjacent passes touch the same handful of files in continuous order with no meaningful boundary between them, consider merging them.

**End-state declarations:** Each pass declares the state the codebase is in at its end:

- `working` — the pass's verification command must succeed before the next pass dispatches. This is the default.
- `broken-intentional` — the pass deliberately leaves the codebase non-working (e.g., it deletes the old code path before a later pass adds the new one). The declaration MUST name the pass that restores working state, e.g., `broken-intentional (restored by Pass 4)`.

Most passes are `working`. `broken-intentional` is for migrations and replacements where the half-state is unavoidable: "delete old auth code → add new auth code → flip callers → restore tests" is naturally 3–4 passes with the middle two non-working. Do not use `broken-intentional` to avoid writing a verification command; use it only when the codebase genuinely cannot pass its checks at the pass boundary.

**Verification:** A `working` pass has a verification command the executor runs between passes — a test, typecheck, lint, or targeted script. The command's exit code is the gate. A `broken-intentional` pass has `Verification: none` because the codebase isn't expected to verify cleanly.

## Task granularity (within a pass)

Each step inside a task is one discrete, verifiable action: one thing the implementer does, followed by something concrete to check. Small steps exist to create checkpoints — they isolate failures, keep each step's output inspectable, and make recovery cheap when something goes wrong. They are not a concession to limited agent capacity.

- "Write the failing test" -- step
- "Run it to verify it fails" -- step
- "Implement the code" -- step
- "Run tests to verify they pass" -- step

Do not bundle independent edits into one step. Frame steps by the action they perform and the check that follows — never by how long they would take, how easy they are, or how much effort they require.

## Plan Header

```markdown
# [Feature Name] Implementation Plan

**Spec:** [Path to the spec this plan implements, e.g., .ok-planner/specs/2026-05-05-foo-design.md]
**Goal:** [One sentence]
**Architecture:** [2-3 sentences]
**Tech Stack:** [Key technologies]

---
```

The `**Spec:**` field is required. The execute-* skills read it to archive
the spec alongside the plan when the work completes. If a plan was
authored without going through `brainstorm` (i.e., no spec exists), write
`**Spec:** none` so the field is still present and explicit.

## Pass and Task Structure

Tasks live under passes. Each pass has a header with goal/end-state/verification; each task under it has its own files list, numbered steps, and verification command. Tasks are numbered globally across the plan, not reset per pass — that way the divergence audit, plan reviewer, and user can reference "Task 7" unambiguously.

```markdown
## Pass 1: Delete the legacy auth path

**Goal:** Remove the old token-based auth code so the new session-based code can land cleanly in Pass 2.
**Scope:** Tasks 1–3
**End state:** broken-intentional (restored by Pass 3)
**Verification:** none

### Task 1: Remove `internal/auth/token.go`

**Files:** `internal/auth/token.go`, `internal/auth/token_test.go`

**Steps:**
1. Delete both files.
2. ...

**Verification:** `git status` confirms the files are gone.

### Task 2: ...

---

## Pass 2: Add session-based auth

**Goal:** Introduce the new session module with full tests.
**Scope:** Tasks 4–6
**End state:** working
**Verification:** `go test ./internal/auth/...`

### Task 4: ...
```

Every verification is a command the implementer can run and interpret from its output. Do not include commit steps, "ask the user" steps, or manual-check steps — the user commits when they are ready and manual checks go in the final section (see Autonomous Execution Required above).

## Plan Review

After writing, dispatch a reviewer:

```
Agent (general-purpose):
  Review the plan at [path] against the spec at [path].

  ## Ground every claim in source code

  Before checking the plan against the spec, verify the plan
  against the codebase. A plan that references files, functions,
  or APIs that do not exist (or have a different shape than the
  plan assumes) sends the implementer chasing ghosts. Catch it
  here.

  - Read every file path the plan names. If a path does not exist
    and the plan does not mark it as new (a "create this file"
    task), flag it.
  - For every function, class, module, type, table, endpoint, or
    other named entity the plan references as existing, find it
    (`rg`, `grep`, or direct read). If it does not exist, or its
    actual shape differs materially from the plan's assumption,
    flag it with file:line and what is actually there.
  - For every "modify X" / "the existing Y" instruction, verify
    the target exists and is shaped as the plan assumes.
  - For every NEW file, type, interface, handler, route, or table
    the plan introduces, find its closest cousin in the codebase
    (sibling files in the same directory, neighboring tables,
    prevailing patterns for similar work) and check that the
    plan's proposed shape and location match. A plan that puts
    new CLI verbs under `cmd/foo/` when every other verb lives in
    `internal/cli/` — or that defines a new persistence table
    with a different transaction-handling convention than its
    neighbors — sends the implementer to build something
    structurally out-of-place. The grounding rule covers
    new-but-conventionally-shaped code, not just references to
    existing entities. If the plan's deviation is intentional
    and the spec justifies it, the plan should say so; if the
    plan is silent, flag it as a grounding miss.

  If you find one such issue, keep looking — they cluster.
  Exhaustively check every file path and every named entity; do
  not sample. The cost of a missed grounding issue is multiplied
  by the cost of the implementer chasing the ghost or building
  the wrong shape.

  ## Plan quality checks

  After grounding, check:
  - Completeness and spec alignment. If the spec has a
    `## Design changes` section, verify the plan contains explicit
    tasks for each bullet (concept file mutations, tension file
    moves to `_resolved/` or `_rejected/`, new concept or tension
    files). Design-doc edits ride alongside code edits in the
    same plan; missing tasks here are blocking.
  - Task decomposition (one discrete action plus verification per
    step)
  - Buildability (the implementer can act on each task from the
    information given)
  - Autonomous executability — every step must be runnable by an
    agent without human intervention. Flag any step that requires
    manual verification, user approval, external environments, or
    credentials the plan does not provide. Manual checks belong
    in a final "Manual checks after completion" section, never
    inside a task.

  ## Pass decomposition checks

  Passes are how `execute-plan` chunks the plan for dispatch.
  Bad decomposition either exhausts a subagent's context or
  fragments the work into too many serial dispatches. Check:

  - Every task lives under a numbered pass (no orphan tasks
    outside any pass).
  - Pass count is appropriate to the plan: most plans are 1–3
    passes; large plans run to many more (a dozen, two dozen, or
    beyond). There is NO upper cap on pass count — it scales with
    the spec's scope, and a high count is never grounds to flag the
    plan as needing to split into multiple plans or to push the
    spec back to be narrowed. Do flag the opposite problems: a
    single pass with 30+ tasks (one subagent shouldn't be asked to
    do that much — it wants splitting into more passes), and passes
    fragmented so trivially that the count is padded.
  - Each pass header declares **Goal**, **Scope** (task
    range), **End state**, and **Verification**.
  - End state is `working` or `broken-intentional`. Every
    `broken-intentional` pass MUST name the pass that restores
    working state (e.g., "restored by Pass 4"). A
    `broken-intentional` pass that doesn't name a restorer is a
    blocking issue — the executor has no signal that the half-state
    is intentional.
  - The final pass MUST end with `working` (the plan can't leave
    the codebase broken). Flag any plan whose last pass is
    `broken-intentional`.
  - `working` passes have a non-empty Verification command the
    executor can run between passes. `broken-intentional` passes
    have `Verification: none`. Flag mismatches.
  - Adjacent passes that touch the same handful of files in
    continuous order with no meaningful boundary — consider
    merging. Passes that enumerate 10+ tasks or scatter across
    unrelated areas — consider splitting.

  Flag any phase/stage/milestone framing aimed at the user (passes
  are internal chunks; "phase 1 of N" framing implies user
  checkpoints and is not allowed). Flag any plan that implements
  only part of the spec — the plan must cover the spec in full
  and execute end-to-end in a single `execute-plan` run. Flag any
  commit, push, branch, PR, deployment, release, or rollout steps —
  plans produce working-tree edits and verification commands;
  delivery process is the user's concern.

  Only flag issues that would cause implementation failure or
  force the executing agent to stall for human input.

  Report: Approved | Issues Found (with specifics)
```

Fix issues, re-review until clean. Each re-review runs the full grounding pass — do not assume the previous round was exhaustive. Cluster bias means missed grounding issues hide behind the ones the previous reviewer found; a second-round reviewer that only verifies the cycle-1 fixes will leave the cluster's tail un-caught.

## Execution Handoff

When the plan is written and the reviewer reports clean, **stop**. Do not invoke any execution skill. Do not start implementing. Do not offer to start implementing in this session.

Tell the user:

- The plan path
- That they should start a **fresh Claude Code session** (clear context) and run `/execute-plan` against the plan there, so the implementer works from a clean context with the plan as its single source of truth

Then end the turn. Plan execution is the user's call, in a session of their choosing.
