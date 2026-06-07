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

## Make load-bearing properties explicit and checkable

A spec usually depends on properties that are easy to state and
easy to lose: durability ("the audit record is canonical"),
completeness ("every event is recorded"), atomicity ("the write
and its index update commit together"), ordering, idempotency,
no-data-loss, "this path must never block the request." These are
the properties that get silently traded away during
implementation — the implementer reaches for an expedient shape
(async, best-effort, sampled, partial) and the plan never told
them the property was load-bearing.

The plan is where a decided property stops being silent. For each
load-bearing property the spec relies on:

- **State it in the implementing task** — name the property and
  the cheaper shape that must NOT be used ("the audit insert is
  synchronous and shares the request's transaction; do not make it
  async or droppable"). The implementer obeys the plan literally,
  so the constraint has to be on the page.
- **Give it a verification command that actually exercises it**,
  not a happy-path check. A property with no real verification can
  regress silently between passes and slip past review. If the
  property is "no row is dropped under load," the verification
  drives load and asserts completeness — not "write one row, read
  it back." If a property genuinely can't be checked by a command,
  state that in the task and add a line to `## Manual checks after
  completion`.

Make these calls **autonomously**, defaulting to the rigorous side
whenever the spec leaves a how-question open — correctness over
speed, durability over latency, completeness over convenience —
exactly as the implementer is told to. Do not stop to ask;
write-plan runs to completion uninterrupted. If you settled a
tradeoff the spec didn't, or promoted an implicit property to an
explicit constraint, note it for the end-of-run handoff. The model
is: make the call, let the reviewer check it, surface it to the
user at the end — not pause mid-plan to ask.

## Proof-first: a behavior change ships as a red test, then the fix

The deepest execution failure is a feature that looks done because a
test is green — but the test never exercised the behavior, so it was
green before the code existed and stays green if the code is deleted.
A green build then certifies a half-wired feature as complete. The
plan prevents this **mechanically**, not by asking the implementer to
"write good tests."

**Every task that changes runtime behavior — a bug fix or a new
feature path — is planned as a red task followed by a green task:**

1. **Red task — add the proof, gate it to FAIL.** Add the test (or
   runnable demo) that drives the *real* system and asserts an
   *observable outcome*. Its verification asserts the test **fails**
   against current code: `Verification: ! <the exact test command>` —
   the gate exits 0 only when the test fails (`!` inverts the exit
   code in any POSIX shell).
2. **Green task — implement the fix, gate it to PASS.** Implement the
   change. Its verification runs the *same named test* and asserts it
   now passes: `Verification: <the exact test command>`.

This is mechanical, not trust. A test that is green against current
code — a struct/proto-shape assertion, a pure-helper call, a test
written after the code — **fails the red gate** (it passes when it
must fail), so `execute-plan` re-dispatches and it cannot reach DONE.
Only a test coupled to the actual behavior passes both gates. The red
task is red-green-revert done forward: the test is *proven* red
without the fix — strictly stronger than reverting after the fact, and
with no risky git surgery during the run.

**What counts as a real proof:**

- **Drives the real system** — the actual handler, the real process, a
  testcontainers-backed store, an end-to-end dispatch — not an
  in-memory fake substituted for the unit under test, and not a pure
  function called in isolation as a proxy for a runtime behavior.
- **Asserts an observable outcome** — a persisted row's state, a call
  made over the wire, a returned status, a terminal state reached,
  bytes written. Not the *shape* of a struct/proto/config, and not
  "returned without error."
- **Named concretely in the red task** — which test function and file,
  what real path it drives, what observable outcome it asserts. "Add a
  test" is not a task. "Add `TestHoldsOnlyAutoTerminal` in
  `runner_test.go` that registers a `holds:`-only template, dispatches
  the co-holder against a testcontainers Postgres, and asserts the
  claim handle reaches `committed`" is.

**The verification names the specific test, not a blanket suite.**
`go test ./...` / `npm test` is satisfied by *any* green test in the
tree, so it can't prove *this* fix. The green task targets the named
test (`go test -run TestHoldsOnlyAutoTerminal ./lib/runtime/...`); a
broad suite run may be added *as well*, never *instead*.

**Red and green are separate tasks, never collapsed** into one "add
code and its test" step. Usually the red task is the tail of a pass
and the green task opens the next, or both sit in one pass as adjacent
tasks. A red task's pass ends `working` with an asserts-failure
verification (`! <test>`) — a legitimate `working` gate, not
`broken-intentional`: the tree builds, one new test is intentionally
red, and the gate confirms exactly that.

**Scope.** The red-*test* requirement applies to behavior changes —
anything that alters what the running system does. Pure
behavior-preserving refactors, doc / concept-doc edits, and mechanical
config changes don't need a red test *file*. They are NOT exempt,
however, from proving their gate flips: every `working` gate — test or
not — is authored as a provable flip (see "Gate flip metadata: every
gate is a provable flip" below). A migration's gate proves "relation
does not exist → exists"; that is a flip even though it is not a test.
In doubt, treat it as behavior and write the red test.

**Fallback (rare).** If a test genuinely can't be written before the
fix (e.g., it can't compile without a type the fix introduces), plan a
green task that adds fix + test, then an immediately following
**revert-check task** that removes only the fix — by editing it out and
back, never with `git stash`/`checkout`/`reset` — runs the test to
confirm it goes red, and restores the fix, capturing the red-then-green
output. Prefer red-first; use this only when red-first is structurally
impossible.

## Gate flip metadata: every gate is a provable flip

A verification command is a real gate only if it is **red when the
pass's work is absent and green when it is present**. `execute-plan`
does not trust a gate on faith — it runs the gate before the work (it
must be red for the *right reason*), runs it after (it must go green),
and if the gate can never pass (a `docker compose run` that starts no
server) or passes proving nothing (a green-from-birth test), it repairs
the gate autonomously and records the repair as a divergence. For the
executor to tell a *right-reason* red (the deliverable is genuinely
absent) from an *infrastructure* red (the command can't even run its
check), every `working` pass authors two lines alongside its
`Verification:`:

- **`Proves:`** — one line naming the *deliverable* a green gate
  establishes ("both new tables and the `profile` column exist and are
  queryable").
- **`Red-when-absent:`** — what the command does on the work-absent
  tree, and the failure signature that means *right-reason* red: a
  failed assertion, a missing relation / column / symbol — NOT an
  infrastructure error (connection refused, command not found, no such
  service).

Authoring `Red-when-absent` is itself a check on the gate: if the only
failure you can describe for the work-absent tree is infrastructural
("it won't connect"), the gate is mis-environed — fix the command, not
the field. For a red task the `! <test>` line already encodes the flip
(`Red-when-absent` is "the named test fails"); state it anyway for
uniformity. `broken-intentional` passes have `Verification: none` and
no flip metadata.

Example — a non-test gate (a schema migration):

```text
**Verification:**     docker compose up -d postgres && docker compose exec -T postgres \
                        psql -U zoning -d zoning -v ON_ERROR_STOP=1 \
                        -c "SELECT FROM data_source_endpoints LIMIT 0; SELECT profile FROM data_sources LIMIT 0;"
**Proves:**           the new table and the profile column exist and are queryable
**Red-when-absent:**  before the migration, psql exits non-zero with
                        `relation "data_source_endpoints" does not exist` — a missing
                        relation (right-reason red), NOT a connection/command error
```

The `run`-vs-`exec` and `ON_ERROR_STOP` details matter: `docker compose
run <svc> psql` starts a throwaway container with no server and can
never connect (a false-negative gate), and a bare `psql -c "\d table"`
exits 0 whether or not the table exists (a false-positive gate). A gate
you cannot give a right-reason `Red-when-absent` is a defective gate —
catch it here, at authoring time.

**Run the gate; don't just describe it.** Authoring `Red-when-absent`
from reasoning is necessary but not sufficient — a command can *read* as
obviously correct and still be green on the work-absent tree, proving
nothing. A described intent is not an observed failure. So for every
`working` gate you can run at authoring time, **execute the verification
command once against the current tree** (which lacks this plan's
deliverables) and read its exit code:

- **Non-zero for the right reason** (a failed assertion, a missing
  relation / column / symbol, a compile error naming the absent
  deliverable) → the red half is proven; write `Red-when-absent` from
  what you actually saw.
- **Exit 0 (green)** → the gate is a **false positive**: it passes while
  the deliverable is absent, so it cannot be coupled to the work. Fix the
  command until it fails for the right reason, then re-run. Never finalize
  a gate you have not seen go red.

Running it is the only way to catch the most common false positive, which
reading the command never reveals: **a test-filter that matches nothing
exits 0.** `swift test --filter X`, `go test -run X`, `pytest -k X`, and
`jest -t X` all *succeed* when the filter selects zero tests, so a gate
naming a not-yet-written suite is green from birth. (Same family as the
`\d`/describe and `docker compose run` traps above — all read fine and
fail only when run.) When the named test does not exist yet, gate on the
test *actually running* — require a non-zero executed-test count, or
assert the suite is listed before running it — so "zero tests matched" is
a failure, not a pass.

**What the dry-run proves, and what it leaves to `execute-plan`.** The
current tree lacks *every* deliverable in the plan, so "is this gate
green when its work is absent?" is answerable now for every `working`
pass — and that false-positive catch is the whole defect class that
otherwise survives to execution. What you cannot finish proving at
authoring time is the *green* half (the gate passes once the work lands)
or the exact right-reason red for a later pass whose work-absent baseline
is "after earlier passes" rather than the current tree — a pass-N gate
may go red against the current tree by failing to compile against
pass-(N-1)'s absent code, which is fine (it is not green). Proving the
full red→green flip against each pass's real work-absent baseline is
`execute-plan`'s job; yours here is to ensure no gate reaches it already
green. A gate whose infrastructure is not available at authoring time (a
device, a service a task stands up, a script a later task writes) cannot
be dry-run — note it and let `execute-plan`'s pre-flight be its first
real run.

## Acceptance: prove every user-outcome story against the real product

Proof-first (above) gates each behavior change with its own red→green
test. But a feature is more than the sum of its behavior changes: the
handler can pass its test, the route can pass its test, and the two can
still never be wired together. Unit and integration tests prove the
parts; nothing there proves the *assembled* feature does its job when the
real product runs. The acceptance pass closes that gap.

**The spec's user-outcome stories are the contract; the plan gates every
one of them.** A spec opens with a complete, enumerated set of
user-outcome stories (see `brainstorm`'s "User outcomes"). The plan's
**final pass(es) are acceptance passes that carry one end-to-end gate per
story** — not one gate for the spec. A spec with five stories has five
acceptance gates; the run cannot complete until all five are green. One
acceptance pass may host several story gates against a single
assembled-product bring-up (cheaper than five separate boots), but every
story gets its own gate and its own observable assertion. Dropping a
story's gate, or merging it into another so its outcome is never
asserted, is the exact hole that ships a promised capability unbuilt.

**Cover each story's necessity closure.** A story's acceptance holds only
if everything *necessary* for its user-outcome exists — including pieces
the spec never spelled out (the proto wiring, the emit site, the
registration that actually does something). The plan must contain tasks
for that closure, not just the literally-named work: trace each story to
the full set of changes without which its acceptance gate stays red, and
plan all of them. The acceptance gate makes this self-checking — a story
whose closure is incomplete cannot pass — but plan the closure up front
rather than discovering it as a red gate mid-run.

Each story's acceptance gate:

- **End state `working`.** Acceptance is the last pass(es), so the plan
  ends on green acceptance gates — every story's.
- **Each gate boots the real assembled product** — the shipped binaries,
  the real image, the real entry point a user would hit (HTTP route, CLI
  verb, wire message) — and drives its story to the story's observable
  outcome. Not an in-process construction of components, not a unit
  harness calling a handler directly: the thing a user actually runs. A
  testcontainers-backed or subprocess-backed end-to-end test that brings
  up the real artifact counts; an in-memory assembly of structs does not.
- **The value-delivering component is real, not stubbed.** If a story
  exists so that some executor, worker, sensor, or subscriber does real
  work, its acceptance gate wires a real one doing real work and asserts
  the real effect. A canned-success stub at this gate defeats the entire
  pass — it is the single thing this pass most exists to forbid.
- **Every gate is proof-first.** Ideally a story's final integrating
  wiring is the green task and its acceptance is authored to FAIL before
  it (`Verification: ! <acceptance command>`), flipping to pass once the
  wiring lands — so the gate cannot pass against an unwired story. Where
  earlier passes already assembled everything and a story's gate is green
  on arrival, use the proof-first **revert-check fallback** (see above) to
  prove it would go red without the story's integrating edit.
- **Each gate names its acceptance command** — runnable and readable from
  its output like every other gate, never a manual "bring up the stack
  and look."

These passes are what make "the product actually works" a machine-checked
gate instead of a `## Manual checks after completion` afterthought. As
specs accumulate, their per-story acceptance gates become the project's
end-to-end regression coverage as a byproduct — committed tests, re-run by
the normal suite — with nothing maintaining a separate corpus.

**When a plan needs no acceptance pass.** Only when the spec states no
user-outcome stories (a pure refactor, docs / concept-doc-only work, an
internal mechanism with no user-reachable surface). If the spec states
any story, the plan must carry an acceptance gate for **each** one; the
planner does not get to drop a story, merge its gate away, or downgrade it
to a unit test.

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

If a feature genuinely requires manual verification that cannot be automated (e.g., visual review of a rendered UI, a human-in-the-loop ML evaluation), **do not** put it inside a task. Collect those items in a final `## Manual checks after completion` section at the end of the plan. That section is for the user to run through after the implementation and code review are done — it is not part of the automated run. **"Does the feature actually work when the product runs" is not one of these items** — that is the acceptance pass (see "Acceptance" above), an automated end-to-end gate, not a manual check. Reserve this section for verifications that genuinely cannot be expressed as a command (true visual judgment of a rendered UI, human-in-the-loop evaluation); never use it to defer end-to-end validation that could be automated.

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

**Verification:** A `working` pass has a verification command the executor runs between passes — a test, typecheck, lint, or targeted script. The command's exit code is the gate. A `broken-intentional` pass has `Verification: none` because the codebase isn't expected to verify cleanly. A verification may *assert failure*: a red-test gate `! <the exact test>` exits 0 only when the named test fails, and is a legitimate `working` verification for a red task (see "Proof-first"). Every `working` gate is also authored with `Proves:` and `Red-when-absent:` so the executor can validate it flips red→green around the work (see "Gate flip metadata: every gate is a provable flip").

## Task granularity (within a pass)

Each step inside a task is one discrete, verifiable action: one thing the implementer does, followed by something concrete to check. Small steps exist to create checkpoints — they isolate failures, keep each step's output inspectable, and make recovery cheap when something goes wrong. They are not a concession to limited agent capacity.

- "Write the failing test" -- step
- "Run it to verify it fails" -- step
- "Implement the code" -- step
- "Run tests to verify they pass" -- step

For a behavior change these steps split across the **red task** and the
**green task** (see "Proof-first"): the red task ends at "run it to
verify it fails" and is gated on that failure (`Verification: ! <test>`);
the green task implements the code and is gated on the test passing. The
"verify it fails" step is not optional or self-reported — it is the red
task's verification command, and `execute-plan` runs it.

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
**Proves:** the new session module exists and its tests pass
**Red-when-absent:** before this pass the package fails to build / has no session tests (a compile or test failure, not an environment error)

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
  - Load-bearing properties are explicit and checked. Identify the
    properties the spec relies on (durability, completeness,
    atomicity, ordering, idempotency, no-data-loss,
    canonical-record, "must not block the request," and the like).
    For each, verify the plan (a) states the constraint in the
    implementing task — naming the cheaper shape that must not be
    used — and (b) gives it a verification command that actually
    exercises the property, not just a happy-path check. A
    load-bearing property with no real verification is a blocking
    gap: it can regress silently during execution and slip past
    review.
  - Proof-first compliance (blocking). For every task that changes
    runtime behavior, the plan must pair a **red task** (adds the
    test, verification asserts it FAILS — `! <the exact test>`) with a
    **green task** (implements the fix, verification runs the *same
    named test* and asserts it passes). Check that: (a) the test
    drives the real system and asserts an observable outcome — a
    persisted row, a wire call, a returned status, a terminal reached
    — NOT a struct/proto/config shape, a pure-helper call, or an
    in-memory fake standing in for the system under test; (b) the red
    task's verification is an asserts-failure gate, not a green-from-
    birth test; (c) the verification names the specific test, not a
    blanket `go test ./...` / `npm test` that any green test satisfies;
    (d) red and green are separate tasks, not collapsed. A behavior
    change with no red task, a shape-only "test", or a blanket-suite
    verification is a blocking gap — that is exactly the defect that
    lets a half-wired feature pass as done.
  - Gate flip metadata (blocking for `working` passes). Every `working`
    pass states `Proves:` and `Red-when-absent:` alongside its
    `Verification:` (see "Gate flip metadata: every gate is a provable
    flip"). Check that: (a) both are present; (b) `Red-when-absent` names
    a *right-reason* failure — a failed assertion, a missing relation /
    column / symbol — NOT an infrastructure error (connection refused,
    command not found, no such service); if the only failure the author
    can name is infrastructural, the gate is mis-environed → flag it;
    (c) the `Verification` command is not a known-defective shape:
    `docker compose run <service> <client-cmd>` (starts no server — use
    `exec` into the running service), a describe / `\d` that exits 0
    whether or not the object exists (use a strict query with
    `ON_ERROR_STOP=1` or equivalent), a test-filter that exits 0 when it
    selects nothing — `swift test --filter X`, `go test -run X`,
    `pytest -k X`, `jest -t X` all *pass* when zero tests match, so a
    filter naming a not-yet-written suite is green from birth (the gate
    must instead require the test to actually run: a non-zero executed
    count, or list-the-suite-then-run) — or a blanket `go test ./...` /
    `npm test` any green test satisfies; (d) **actually run each
    `working` gate's `Verification` command against the current tree and
    read its exit code** — do not judge the gate by reading it alone. The
    current tree lacks the plan's deliverables, so a gate that exits 0
    here is a false positive (it proves nothing about work that does not
    yet exist) and is blocking no matter how correct the command reads;
    a non-zero exit for a right reason (failed assertion / missing
    symbol / compile error naming the absent deliverable) is fine, while
    a non-zero exit only from an infrastructure error is mis-environed →
    flag it. Skip the run only for a gate whose infrastructure genuinely
    is not available at authoring time (a device, a service the plan
    stands up, a script a task writes) — and note which gates you could
    not run. A `working` gate missing its flip metadata, carrying an
    infrastructure-only `Red-when-absent`, using a known-defective
    command, or exiting 0 on the work-absent tree is a blocking gap —
    `execute-plan` can repair some of these at runtime, but a gate proven
    red at authoring time is cheaper and never burns a run.
  - Acceptance-pass compliance (blocking). If the spec states
    user-outcome stories, the plan's final pass(es) must carry **one
    acceptance gate per story** — not one gate for the whole spec.
    Check (1) coverage: every story has its own acceptance gate that
    asserts that story's observable outcome; a story with no gate, or
    whose gate is merged into another so its outcome is never
    asserted, is a blocking gap — that is exactly how a promised
    capability ships unbuilt. Then, for each gate: (a) it boots the
    real assembled product / shipped entry point — not an in-process
    construction of components or a unit harness calling a handler
    directly; (b) it drives its story to the story's real observable
    outcome; (c) it keeps the value-delivering component real — a real
    executor / worker / sensor / subscriber doing real work; a stub or
    canned-success substitute at this gate is a blocking gap; (d) it is
    gated proof-first (red-then-green, or the revert-check fallback) so
    it cannot pass against an unwired story; (e) it names a runnable
    acceptance command, not a manual check. Also check **necessity
    closure**: for each story, the plan contains the tasks without
    which its acceptance gate could not go green — including pieces the
    spec did not literally name (proto wiring, emit sites, the
    registration that does real work). A story whose closure is
    under-planned will stall at a red acceptance gate during execution;
    flag a visibly incomplete closure here. A spec with no user-outcome
    stories (pure refactor / docs-only / no user-observable change)
    correctly has no acceptance pass; do not flag its absence there.
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
    `broken-intentional`. For a spec that states an acceptance
    scenario, this final `working` pass is the acceptance pass (see
    "Acceptance-pass compliance" above).
  - `working` passes have a non-empty Verification command AND the
    `Proves:` / `Red-when-absent:` flip metadata (see "Gate flip
    metadata" under Plan quality checks). `broken-intentional` passes
    have `Verification: none` and no flip metadata. Flag mismatches.
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
- Any autonomous calls worth knowing about: load-bearing properties the plan made explicit and checkable, and any tradeoff the spec left open that the plan defaulted to the rigorous side of. Keep this to a short list — these are calls already made, surfaced for awareness, not questions to answer. If there were none, omit this entirely.

Then end the turn. Plan execution is the user's call, in a session of their choosing.
