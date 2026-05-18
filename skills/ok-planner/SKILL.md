---
name: ok-planner
description: "ONLY activated by explicit slash command (/brainstorm, /write-plan, /review, /discover-design, /refine-design, etc). Never auto-triggered by conversation content."
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

## Instruction Priority

1. **Project rules** (`.claude/rules/`) -- highest priority, non-negotiable
2. **User's explicit instructions** (CLAUDE.md, direct requests)
3. **ok-planner skills** -- override default system behavior where they conflict
4. **Default system prompt** -- lowest priority

Project rules and user instructions always win. If rules.md or CLAUDE.md contradicts a skill, follow the rules.

## Available Skills

Invoke via the `Skill` tool with the `ok-planner:` prefix.

| Skill | When to use |
|-------|-------------|
| `ok-planner:init` | **Internal.** Invoked by other ok-planner skills before they produce or move artifacts. Ensures `.ok-planner/{specs,plans,sketches,design,history/specs,history/plans}/` exists. Idempotent. Safe for the user to invoke directly via `/init`. Use `/init --refresh` to update an existing `.ok-planner/CLAUDE.md` to the current template (with diff + confirmation). |
| `ok-planner:discover-design` | User types `/discover-design`. Runs autonomously end-to-end via produce → review → fix loops. Two phases: (1) reads code + prose and writes as-is scaffolding to `.ok-planner/design/_discover/`; (2) extracts load-bearing concepts to `.ok-planner/design/concepts/`, a tensions catalog to `.ok-planner/design/tensions/`, and agent-confessed uncertainty to `.ok-planner/design/review-notes.md`. Outputs are as-is, not prescriptive. Aborts rather than overwrite human-edited `concepts/` or `tensions/`. |
| `ok-planner:refine-design` | User types `/refine-design`. Specialization of `brainstorm` for resolving design tensions. Read-only intake — user picks tensions and resolution shapes; decisions go into a brief — then hands off to `brainstorm` to produce a spec with a `## Design changes` section capturing the concept-doc mutations and tension moves, alongside the code reconciliation. Flows through `write-plan` → `execute-plan`. |
| `ok-planner:merge` | User types `/merge`. Surfaces design-doc findings against a recent merge or rebase (mechanical issues, drift, semantic conflicts) and runs a code review of the merge diff. Read-only against `.ok-planner/design/` — findings are written to a transient report and the user takes them through `/brainstorm` or `/refine-design` to produce a reconciliation spec. |
| `ok-planner:sketch` | User types `/sketch`. Single-pass design sketch for a new feature, saved to `.ok-planner/sketches/`. Pre-spec, no review loop, no dialogue. Does not lead into write-plan. |
| `ok-planner:brainstorm` | User types `/brainstorm` |
| `ok-planner:write-plan` | You have an approved spec and need an implementation plan |
| `ok-planner:execute-plan` | You have a plan and want a single subagent to drive it to completion, then review |
| `ok-planner:review-work` | Review all uncommitted changes |
| `ok-planner:review-plan` | Review implementation against a spec/plan (interactive: lists plans) |
| `ok-planner:review-commits` | Review a commit range (interactive: shows history) |
| `ok-planner:review-files` | Deep review of specific files/directories (interactive: shows structure) |
| `ok-planner:review-feature` | Exploratory review of a feature area (interactive: asks questions) |
| `ok-planner:review-holistic` | Codebase-wide convergence review aimed at parsimony — finds real issues plus extraneous accretion (code/tests/features/docs that don't earn their keep). Use after large deliveries or when other review cycles have plateaued. Run until convergence. |
| `ok-planner:review-cleanup` | **Internal.** Invoked by review skills after issues are found. Drives the fix-review loop via subagent. Do NOT invoke directly — the review skills call it. |
| `ok-planner:verify` | You're about to claim work is complete |

## Artifact layout

All ok-planner skills read and write under `.ok-planner/` at the project
root (created on demand by `ok-planner:init`):

- `.ok-planner/specs/` — active specs from `/brainstorm` (including specs produced via `/refine-design`'s handoff to brainstorm)
- `.ok-planner/plans/` — active plans from `/write-plan`, plus the
  `-divergences.md` reports written by `/execute-plan`'s divergence
  auditor
- `.ok-planner/sketches/` — design sketches from `/sketch`
- `.ok-planner/design/` — durable design docs (bootstrapped by
  `/discover-design`; mutated only by `/execute-plan` carrying out
  spec-directed plan tasks). Layout: `_discover/` (as-is
  scaffolding), `concepts/` (load-bearing nouns with definitions +
  boundaries + invariants), `tensions/` (catalog of muddy /
  unspecified / conflicting bits).
- `.ok-planner/history/specs/` and `.ok-planner/history/plans/` —
  archived specs and plans moved here automatically when an execute-*
  skill finishes a plan

Specs / plans / sketches / history are workflow scratch, not project
documentation; do not propose updating them to reflect codebase
evolution. The design docs under `design/` are the exception: they
are durable and have the same source-of-truth weight as code,
updated alongside the code through `/execute-plan` runs.

## When Skills Activate

**ok-planner skills are NOT auto-triggered.** They activate when:
- The user explicitly types a slash command (e.g., `/brainstorm`, `/review`)
- A skill's output directs you to invoke the next skill in the pipeline (e.g., brainstorm leads to write-plan)

Do NOT invoke skills based on inference about what the user might want. Wait for the slash command.

## Model Selection

Always use the most capable model available. Do not downgrade models for "simple" tasks. The user pays for quality, not savings.

## Code Review: Fix Everything

When processing code review results: **fix every issue found, regardless of severity label or origin.** Do not triage. Do not defer. Do not note for later. The reviewer may categorize issues as Critical/Important/Minor -- ignore the categories and fix them all. This overrides any skill instruction that suggests deferring minor issues.

This includes pre-existing issues discovered during review. If the reviewer reports something was "already stale," "predates this work," or "out of scope," fix it anyway. If fixing a discovered issue requires pulling additional files into the review, expand the scope. Leaving known issues unfixed because they weren't part of the original task is never acceptable.

**Structural enforcement:** All review skills invoke `ok-planner:review-cleanup` after receiving reviewer output. This skill dispatches a fixer subagent and re-reviews until clean, removing the orchestrator's ability to triage. If you find yourself writing "acceptable," "noted for later," "cosmetic," or "out of scope" next to an unfixed issue — you are violating this rule. Stop and invoke review-cleanup instead.
