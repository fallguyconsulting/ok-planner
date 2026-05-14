---
name: write-plan
description: "ONLY activated by explicit /write-plan slash command or ok-planner pipeline. Never auto-triggered."
---

# Writing Plans

Write implementation plans assuming the implementer has zero codebase context. Document: which files to touch, exact code, tests, verification commands. Bite-sized tasks.

**Save plans to:** `.ok-planner/plans/YYYY-MM-DD-<feature-name>.md` (user preferences override)

**Before writing:** invoke `ok-planner:init` to ensure `.ok-planner/plans/` exists.

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

## The plan captures the spec, in full

The spec is the unit of work. The plan's job is to translate it into executable
tasks — not to revise its scope. Whatever the spec covers, the plan covers, end
to end, in a single run.

If the spec has a `## Tensions resolved` section (the marker that this work
came in via `refine-design`), treat the concept-file mutations and the
tension-file moves it specifies as first-class plan tasks alongside the code
work. Each `concepts/<slug>.md` mutation gets explicit edit instructions; each
resolved tension gets a task to move `tensions/<slug>.md` to
`tensions/_resolved/<slug>.md` with `status: resolved` and a `resolution`
block summarizing the outcome. Design-doc edits are not optional follow-up —
they ship in the same plan as the code that conforms to them.

## Design log awareness

If `.ok-planner/design/concepts/` exists, it is the project's
canonical concept catalog. To find what applies to the files the
plan will touch:

1. Read `.ok-planner/design/concepts.md` — the auto-generated TOC.
2. Grep for `@concept:` annotations in the files the plan will
   touch (`rg '@concept:' <path>`).
3. Read `concepts/<slug>.md` in full for each concept surfaced by
   step 1 or 2 that's load-bearing for the plan.

Plan tasks must respect each consulted concept's stated boundaries
and invariants:

- If a task would violate an invariant or drift from a stated
  boundary without the spec's `## Tensions resolved` section
  authorizing it, surface that to the user before writing the plan —
  it means either the spec missed a design impact or the concept
  doc needs updating. Don't silently plan an invariant-violating
  change.
- When tasks touch files claimed by a concept, cite the concept
  briefly in the task notes ("respects boundary X from
  `concepts/<slug>.md`") so the implementer reads it. If a task is
  the first one to touch a load-bearing site for a concept that
  doesn't yet have an `@concept:` annotation there, include adding
  the annotation as part of the task.

If `.ok-planner/design/concepts/` doesn't exist, this skill operates
as it does today. Do not create it as a side effect.


- No phases, stages, milestones, or "phase 1 of N." A plan is executed start to
  finish in one run.
- No partial-spec plans. If the spec describes ten unrelated changes, the plan
  contains tasks for all ten. Cohesion, independence, and "is this really one
  feature" are not the planner's concerns — those decisions were made when the
  spec was written.
- No commit, push, branch, or PR steps. The user owns git. Plans produce
  working-tree edits and verification commands; nothing else.

If the spec is genuinely unworkable (contradictory, missing required context,
references things that do not exist), surface that to the user before writing
the plan. Do not silently narrow the scope.

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

## Task Granularity

Each step is one discrete, verifiable action: one thing the implementer does, followed by something concrete to check. Small steps exist to create checkpoints — they isolate failures, keep each step's output inspectable, and make recovery cheap when something goes wrong. They are not a concession to limited agent capacity.

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

## Task Structure

Each task has: files list, numbered steps with code, verification commands. Every verification is a command the implementer can run and interpret from its output. Do not include commit steps, "ask the user" steps, or manual-check steps — the user commits when they are ready and manual checks go in the final section (see Autonomous Execution Required above).

## Plan Review

After writing, dispatch a reviewer:

```
Agent (general-purpose):
  Review the plan at [path] against the spec at [path].
  Check: completeness, spec alignment, task decomposition, buildability, and
  autonomous executability — every step must be runnable by an agent without
  human intervention. Flag any step that requires manual verification,
  user approval, external environments, or credentials the plan does not
  provide. Manual checks belong in a final "Manual checks after completion"
  section, never inside a task.
  Flag any phase/stage/milestone framing or any plan that implements only
  part of the spec — the plan must cover the spec in full and execute
  end-to-end in a single run. Flag any commit, push, branch, PR, deployment,
  release, or rollout steps — plans produce working-tree edits and
  verification commands; delivery process is the user's concern.
  Only flag issues that would cause implementation failure or force the
  executing agent to stall for human input.
  Report: Approved | Issues Found (with specifics)
```

Fix issues, re-review until clean.

## Execution Handoff

When the plan is written and the reviewer reports clean, **stop**. Do not invoke any execution skill. Do not start implementing. Do not offer to start implementing in this session.

Tell the user:

- The plan path
- That they should start a **fresh Claude Code session** (clear context) and run `/execute-plan` against the plan there, so the implementer works from a clean context with the plan as its single source of truth

Then end the turn. Plan execution is the user's call, in a session of their choosing.
