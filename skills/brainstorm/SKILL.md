---
name: brainstorm
description: "ONLY activated by explicit /brainstorm slash command. Never auto-triggered by conversation content."
---

# Brainstorming Ideas Into Designs

Turn ideas into specs through collaborative dialogue. Understand context, ask questions, present design, get approval.

<HARD-GATE>
Do NOT write code, scaffold, or take implementation action until the user approves a design. The design can be short for simple projects, but it must exist and be approved.
</HARD-GATE>

<DIALOGUE-REQUIRED>
This skill is a dialogue with the user. The clarifying questions, section-by-section approvals, spec review, and handoff to write-plan are not optional. Specifically, do not:

- Collapse the intake into assumptions and proceed
- Bundle multiple questions into one message — one question per message, wait for the answer
- Pick between options you have offered the user on their behalf — present the options, then wait
- Move past a question without a user answer in hand
- Chain into `write-plan` before the user has approved the written spec

The product of this skill is a user-approved spec — that requires the user. General "run unsupervised" guidance does not apply here; this skill *is* the dialogue.
</DIALOGUE-REQUIRED>

## Process

1. **Initialize layout** -- invoke `ok-planner:init` so `.ok-planner/specs/` exists before any writing starts.
2. **Establish the goal** -- if context from the conversation makes it clear what to brainstorm, proceed. Otherwise ask the user what they want to design. If invoked from `ok-planner:refine-design`, the goal has already been established in the conversation (a brief naming the tensions to resolve, the picked resolution shapes, the affected concept files, and the code paths needing reconciliation) — recognize that brief as the goal and proceed.
3. **Explore project context** -- check files, docs, recent commits
4. **Ask clarifying questions** -- one at a time, prefer multiple choice
5. **Propose 2-3 approaches** -- with trade-offs and your recommendation
6. **Present design** -- scaled to complexity, get user approval section by section
7. **Write spec** -- save to `.ok-planner/specs/YYYY-MM-DD-<topic>-design.md`
8. **Spec review** -- dispatch reviewer subagent, fix issues, re-review until clean
9. **User reviews spec** -- ask user to review before proceeding
10. **Update design log** -- if the spec has a `## Tensions resolved` section (refine-design-originated), skip; the impacts are already recorded. If `.ok-planner/design/concepts/` does not exist, skip silently — the design log is opt-in, and projects that haven't bootstrapped one are not waiting to be told about it. Otherwise, walk candidate concept-impacts and tension-impacts with the user and apply them to `.ok-planner/design/concepts/` and `.ok-planner/design/tensions/` per `discover-design`'s templates. Skip entirely if the spec produced no design impacts.
11. **Transition** -- invoke ok-planner:write-plan

## What a spec is

A brainstorm produces **one spec**, capturing the full scope the user is
designing. Do not split into multiple specs. If the scope is broader than one
brainstorm can cover well, surface that to the user — they decide whether to
narrow the brainstorm or keep going.

A spec describes **what to build**: architecture, components, data flow,
behavior, error handling, testing strategy. It does not describe **how to
ship it**. No PRs, commits, branches, merge strategies, deployment steps,
release plans, or rollout phases. Delivery process is out of scope — the
user owns that.

## Asking Questions

- One question per message
- Prefer multiple choice when possible — presented as plain text (A, B, C…) with a recommendation. Free-form replies are always fine.
- Do **not** use tools that constrain the user's reply to a predetermined structure (e.g. `AskUserQuestion`, poll-style pickers, forced single-select widgets). Ask in prose so the user can answer in prose, pick a letter, push back, reframe the question, or go off-script.
- Focus on purpose, constraints, success criteria
- Scope is the user's call. If it feels unclear or sprawling, ask — do not unilaterally decompose into sub-projects.

## Presenting the Design

- Scale each section to its complexity
- Ask after each section whether it looks right
- Cover: architecture, components, data flow, error handling, testing
- YAGNI ruthlessly

## Design log awareness

If `.ok-planner/design/concepts/` exists, it is the project's
canonical concept catalog. During step 3 (Explore project context):

1. Read `.ok-planner/design/concepts.md` — the auto-generated TOC,
   always small, always one-shot-readable. Now you know what
   concepts exist.
2. Grep for `@concept:` annotations in the area you're exploring
   (`rg '@concept:' <path>`). Each marks a load-bearing site.
3. Read `concepts/<slug>.md` in full for any concept surfaced by
   step 1 or 2 that the brainstorm's subject area touches.

Use the catalog's terms in the design, respect its stated boundaries
and invariants, and surface as a concept-impact (or tension-impact)
anything the design would change about a concept's shape.

Also check `.ok-planner/design/tensions/` for open tensions in the
same area — they may already describe the muddiness the brainstorm
wants to address, in which case the user might want to run
`/refine-design` for those tensions instead of (or alongside) this
brainstorm.

If neither directory exists, this skill operates as it does today.
Do not create them as a side effect; do not suggest
`/discover-design` unless the user asks.

## Working in Existing Codebases

- Explore current structure before proposing changes. Follow existing patterns.
- Where existing code has problems that affect the work, include targeted improvements
- Don't propose unrelated refactoring

## After Approval

Write the spec to `.ok-planner/specs/YYYY-MM-DD-<topic>-design.md` (user preferences override this path). Dispatch a spec reviewer subagent:

```
Agent (general-purpose):
  Read the spec at [path]. Check for:
  - Completeness (TODOs, placeholders, incomplete sections)
  - Internal consistency (contradictions, conflicting requirements)
  - Clarity (ambiguity that could cause wrong implementation)
  - YAGNI (unrequested features)
  Flag any PR, commit, branch, deployment, release, or rollout instructions
  — specs cover what to build, not how to ship it. Flag any indication that
  the work has been split across multiple specs.
  Only flag issues that would cause real problems during implementation.
  Report: Approved | Issues Found (with specifics)
```

Fix issues, re-review until clean. Then ask user to review the written spec.

## After User Approval — Update the Design Log

Specs frequently touch the project's concept catalog or surface new
questions about a concept's boundary. The durable design log under
`.ok-planner/design/` is the place to record those touches.
Reviewers consult that log to distinguish design choices from
defects, so capturing what a spec says about concepts is what gives
the log its convergence value.

**If the spec contains a `## Tensions resolved` section** (the marker
that this brainstorm was invoked via `refine-design`): the
design-impact decisions have already been made by the user during
refine-design's intake and are recorded in that section. Skip the
walkthrough below — execute-plan applies the recorded mutations.
Continue to the Transition step.

For any other spec, walk it section by section and identify two
kinds of impact:

**Concept impacts** — the spec sharpens, splits, merges, or
introduces a concept. Candidate belongs as a concept update if it
meets ANY of these:
- It introduces a new load-bearing noun the project did not have.
- It changes the boundary of an existing concept (what's in vs.
  what's now in a neighbor).
- It retires a concept, or retires an alias that should no longer be
  used.

**Tension impacts** — the spec surfaces unresolved questions about
how concepts should fit together. Candidate belongs as a tension if
it meets ANY of these:
- It describes a choice between alternatives that the spec itself
  doesn't resolve (left for later).
- It exposes inconsistency or overloading the team had not yet
  catalogued.
- It declares a tradeoff explicitly but acknowledges the tradeoff is
  not fully understood.

A candidate does NOT belong if it's a purely local implementation
detail. The bar: would a reviewer benefit from knowing this when
forming findings?

If `.ok-planner/design/concepts/` does not exist, skip this whole
section silently. The design log is opt-in; brainstorm works fine
without it, and projects that haven't bootstrapped one are not
waiting to be told about it. Do not mention the absence to the user
or suggest `/discover-design` unless they ask about it. Do not
create the directory as a side effect — bootstrapping is its own
walkthrough.

For each concept candidate:

1. Walk it with the user — confirm-as-is / edit / reject.
2. If accepted and it represents a resolved change (the spec is
   prescriptive about the concept), mutate the relevant
   `.ok-planner/design/concepts/<slug>.md` in place. Append a Notes
   entry citing the spec slug. If the concept doesn't exist yet,
   write a new file using the template defined in
   `ok-planner:discover-design`'s SKILL.md (Concept template).
3. If accepted but the change is still tentative, write a tension
   entry instead (next paragraph).

For each tension candidate:

1. Walk it with the user — confirm-as-is / edit / reject.
2. If accepted, write a new file at
   `.ok-planner/design/tensions/<slug>.md` using the Tension
   template documented in `ok-planner:discover-design`'s SKILL.md
   (frontmatter with `tension` / `category` / `status: open` /
   `affects`, plus What is muddy / Why it matters / Resolution
   candidates / Evidence sections).

If the spec produced no concept or tension impacts (small specs
often won't), say so explicitly and skip to the transition step
rather than padding with placeholders.

## Transition

Once the spec is approved AND any design log entries are written,
invoke ok-planner:write-plan.
