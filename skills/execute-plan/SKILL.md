---
name: execute-plan
description: "ONLY activated by explicit /execute-plan slash command or ok-planner pipeline. Never auto-triggered. Drives a plan to completion pass by pass as a Workflow with code gates: dispatch the implementer, judge a structured report, observe whether each pass's verification gate flips red→green around the work, autonomously repair a defective gate (validated by the flip, never weakened), converge every pass, then run the divergence audit. Human-facing intake, escalation, final review, and the divergence walk stay in the skill."
---

# Executing a Plan (Workflow engine)

Drive a plan to completion pass by pass, then audit, review, and walk
divergences. The **control flow is code, not prose**: the dispatch /
judge / gate / repair / re-dispatch / converge loop runs inside a
`Workflow` whose branches are real conditions on structured agent
output, so a pass cannot advance on a verification the orchestrator
"decided" was close enough.

**Why code gates.** A prose orchestrator can skip a gate, rationalize a
DONE, mis-count re-dispatches, or advance on a green-but-shallow test.
The workflow's control flow can't: `if (!gate.exitZero) redispatch()` is
code, and the exit code it branches on is read by a **separate** agent
whose only job is to report the boolean — so the agent that wants the
pass to advance is no longer the one judging whether it may.

**Gate validity: observe the flip.** The exit code is only trustworthy
if the gate command is *valid* — red when the pass's work is absent,
green when it is present. A command that can never pass (a `docker
compose run` that starts no server) or that passes proving nothing (a
green-from-birth test) is a *defective gate*, and the old failure mode
was to blame the implementer for it, burn the retry cap, and escalate a
human for a one-line fix. This engine instead **observes the flip**:

- A valid gate is red-because-not-built (flips to green when work lands),
  red-because-work-wrong (flips when the work is fixed), or — the case to
  catch — red-because-the-command-is-defective (never flips).
- The engine never classifies the red up front. It watches whether
  completing the work flipped the gate. If the deliverable is present and
  the gate is *still* red, the **gate** is the suspect, not the
  implementer.
- A defective gate is repaired by a **disinterested** agent (never the
  implementer, which has a stake in advancing), and the repair is adopted
  **only if it itself flips** — red against the work-absent state,
  green against the work-present state. A rubber stamp is green in both
  states, fails the flip test, and is rejected, so the repair cannot
  weaken what the gate proves. Autonomous gate repairs are **logged as
  divergences**, surfaced to the user after the run, never as a blocker.

The proof-first red/green discipline from `write-plan` rides along: a
red task's `! <test>` verification is exit-0 only when the named test
fails, and is left to gate exactly as written (red gates skip pre-flight
repair — being green-when-the-test-is-absent is their whole point).

## What runs where (the seam)

| In the **workflow** (code, headless) | In the **skill** (prose, human-facing) |
|---|---|
| dispatch one implementer agent per pass | parse the plan → structured passes |
| judge the report (schema); a BLOCKED is re-checked by a skeptic | the escalation conversation (genuine BLOCKED, work-stuck, gate-undecidable, invalid plan) |
| pre-flight + classify the gate; run it post-work; route by who's at fault | the final review (`review-work` → `review-cleanup`) |
| disinterested probe / diagnosis / repair / flip-validation of a defective gate | the divergence walk (incl. adopted gate corrections) + rework decisions, archive |
| advance or re-dispatch (implementer cap 3; gate-repair cap 2); converge; divergence audit | |

Every role in the engine except the implementer is **disinterested** —
none has a stake in advancing the pass. The implementer is the only role
that wants the pass to advance, and it touches none of the gate
machinery.

The **final review** stays skill-side (reuse `review-work` /
`review-cleanup` as-is) and **resume across an escalation** stays
skill-side (a fixed BLOCKED is resumed by re-invoking the workflow on the
remaining passes). Moving the review-to-clean loop into the workflow is a
natural v2; so is multi-session durability (rimsky's domain). Note them;
don't build them here.

## Opt-in

Invoking the `Workflow` tool from this skill is authorized: the user ran
`/execute-plan` (or the ok-planner pipeline reached it), and this skill's
instructions tell you to call it. That is a valid Workflow opt-in. Pass
the structured passes as `args`; do not paste the plan text into the
script. (For a plan too large for the inline `args` channel — it can drop
or stringify a big payload — embed the passes in the script instead, as
the engine's input guard notes.)

## Process

1. **Select the plan.** If the user named a plan, use it. Otherwise list
   `.ok-planner/plans/*.md` (newest first, filename + title line) and ask
   which to run. Read it in full. Read the `**Spec:**` header.

2. **Parse the plan into structured passes.** For each pass capture an
   object:
   ```
   { number, name, goal, endState,            // "working" | "broken-intentional"
     restorer,                                  // pass number, if broken-intentional
     verification,                              // command string, or "none"
     isRedGate,                                 // true if verification.trim() starts with "!"
     proves,                                    // the **Proves:** line: what a green gate establishes (or "")
     redWhenAbsent,                             // the **Red-when-absent:** line: the right-reason failure signature (or "")
     tasks: [ { number, text } ] }              // task text VERBATIM from the plan
   ```
   `proves` and `redWhenAbsent` come from the gate block `write-plan`
   now authors alongside `Verification:` for every `working` pass. They
   are what let the engine tell a *right-reason* red (the deliverable is
   genuinely absent) from an *infrastructure* red (the command can't run
   its check). A plan authored before this convention has neither — pass
   `""` for both; the engine degrades to best-effort defect detection
   (it can still flag a green or an obvious infra-error pre-flight, and
   still flip-validates any repair).

   If the plan has no passes at all (an older flat-task plan),
   synthesize a single pass — `number: 1`, `name` from the plan title,
   `endState: "working"`, the plan's overall verification command (or
   `none` if it has none), `isRedGate` set accordingly, empty `proves`/
   `redWhenAbsent`, and every task — and tell the user in one line before
   running that the plan predates the pass model and is being executed as
   a single pass.

   Validate before running: every `broken-intentional` pass names a
   restorer that exists; the final pass is `working`; `working` passes
   have a non-empty verification (the exception is the synthesized legacy
   single pass above, which may be `none`), `broken-intentional` passes
   have `none`. If the plan is internally inconsistent, **stop and
   surface to the user** — that is a planner defect, not something the
   workflow should run into. (The engine independently re-checks the two
   invariants that guarantee the tree ends working — final pass
   `working`, every restorer present — and returns an `invalid-plan`
   escalation if either is violated, see step 5; catching the rest here
   keeps it out of a workflow run.)

3. **Affirm the layout.** Invoke `ok-planner:affirm`.

4. **Run the engine as a workflow.** Call `Workflow` with the script in
   "The workflow engine" below and:
   ```
   args = { passes: <the parsed pass array>,
            planPath: "<plan path>",
            specPath: "<spec path from the **Spec:** header, or 'none'>",
            priorCleared: 0 }   // passes cleared in earlier runs; 0 on a first run, raised on resume (step 5)
   ```
   The workflow runs in the background; you are re-invoked when it
   completes. Its return value is a result object — act on its `status`.

5. **Handle the result.**
   - `status: "complete"` → the workflow cleared every pass and produced
     the divergence report. Surface its path + one-line count to the
     user, and (if `gateCorrections` is non-empty) a one-line note that
     N defective gate(s) were autonomously repaired and recorded in the
     report. Then go to step 6.
   - `status: "escalate"` → the workflow stopped at a boundary only a
     human can clear. Surface it verbatim and **wait**:
     - `kind: "blocked"` — a BLOCKED the engine's skeptic confirmed
       genuine (bullet 1 credentials/access, or bullet 2 unauthorized
       destructive action) on `pass`, or one it could not adjudicate.
       Misapplied BLOCKEDs never reach you — the engine re-dispatches
       those itself. Give the user the bullet, the detail, and the
       passes already cleared.
     - `kind: "work-stuck"` — the implementer ladder is exhausted on
       `pass` without a clean DONE that satisfies a *valid* gate (the gate
       flips; the work genuinely can't pass it). The ladder is not bare
       repeats: attempt 1 is the implementer, attempt 2 adds the specific
       failure, and before attempt 3 a **fresh-perspective strategist**
       diagnoses the root cause and prescribes a *different* approach — so
       reaching this escalation means even a different angle failed, i.e.
       genuinely-new input is needed. This means the work is wrong, not
       the gate (a defective gate would have been repaired, not counted
       here). The result is **decision-ready**: `lastFailure` carries the
       strategist's root-cause read and the different approach already
       tried (and, if the strategist judged it, `rootCause` + `humanAsk`).
       Surface it with the options: answer the `humanAsk` / sharpen the
       brief and resume, rework the pass/plan, or abandon.
     - `kind: "gate-undecidable"` — `pass`'s gate is defective and no
       disinterested repair produced a command that flips (red without
       the work, green with it). The gate's intent can't be expressed as
       a runnable flip — a real planner/design defect. Surface the
       `detail` (the original command and why repair failed) and take it
       back to `write-plan` (or fix the gate inline). Do not resume until
       the gate is fixed.
     - `kind: "invalid-plan"` — the engine's defensive validation found a
       passes structure that can't end in a working tree (final pass not
       `working`, or a `broken-intentional` pass naming a restorer that
       isn't in the plan). This should have been caught in step 2;
       surface the `detail` and the offending `pass` and take it back to
       `write-plan`. Do not resume until the plan is fixed.
     When the user resolves a `blocked`, `work-stuck`, or
     `gate-undecidable`, **resume** by re-invoking the workflow (step 4)
     with `args.passes` set to the remaining passes (the stuck pass
     onward) and `args.priorCleared` raised by this escalation's
     `cleared` count (so the final `passesCleared` total stays accurate
     across runs). A fresh run from the stuck pass is correct because the
     fix changed the tree or the plan; do not journal-resume across a
     human fix.

6. **Final review (skill-side).** Invoke `ok-planner:review-work` for the
   full implementation; let `review-cleanup` drive the fix cycle to
   clean. (Reused unchanged. The workflow already ran the proof-first
   gates pass-by-pass; this is the holistic pass.)

6a. **Regenerate `concepts.md`** if any pass touched
   `.ok-planner/design/concepts/`.

7. **Walk the divergence report with the user** — every entry. Adopted
   gate corrections appear there too; a gate the engine had to repair is
   worth the user's eye even though the run succeeded (it means the plan
   authored a defective gate). Reworks come from this conversation.

8. **Archive** the plan, its `-divergences.md`, and the linked spec into
   `.ok-planner/history/`.

## The workflow engine

Pass this as the `Workflow` `script`. It is plan-agnostic — it reads
everything from `args`. Each `working` pass is gated by observing the
flip: pre-flight the gate (it must be right-reason red before the work),
dispatch the implementer, run the gate again (green ⇒ it flipped ⇒
advance), and if it is still red after a credible DONE, probe whether the
deliverable exists, diagnose work-vs-gate, and repair the gate only with
a correction that itself flips.

```javascript
export const meta = {
  name: 'execute-plan-engine',
  description: 'Deterministic, flip-gated pass-by-pass execution engine for an ok-planner plan: dispatch the implementer, judge a structured report, observe whether each pass verification flips red→green around the work, autonomously repair a defective gate (adopted only if the repair itself flips), re-dispatch up to a cap, converge every pass, then run the divergence audit.',
  phases: [
    { title: 'Execute', detail: 'one implementer per pass, flip-gated' },
    { title: 'Audit', detail: 'divergence auditor over the whole diff' },
  ],
}

// Input guard. The engine assumes `args` is the parsed passes object the skill
// hands in, but the Workflow `args` channel can deliver a large inline payload
// as a JSON *string* (or, if dropped, as undefined) rather than a parsed object
// — and the stock engine's first use of `args.passes` then dies on a cryptic
// `undefined is not an object (evaluating 'passes.length')`. Normalize a string
// payload by parsing it, then fail LOUD with a diagnostic if `passes` still
// isn't a non-empty array, so a delivery failure is legible at the boundary
// instead of crashing on an undefined access downstream. (This guards input
// *arrival/shape*; the per-pass `invalid-plan` checks below guard plan
// *semantics* — both are needed.)
let input = args
if (typeof input === 'string') {
  try {
    input = JSON.parse(input)
  } catch (e) {
    throw new Error(`execute-plan-engine: args arrived as an unparseable string (${e.message}); expected the structured passes object { passes:[…], planPath, specPath }.`)
  }
}
if (!input || typeof input !== 'object' || !Array.isArray(input.passes) || input.passes.length === 0) {
  const shape = input && typeof input === 'object'
    ? `an object with keys [${Object.keys(input).join(', ')}]`
    : `a ${typeof args} value`
  throw new Error(`execute-plan-engine: no passes to run — args resolved to ${shape}. The skill must hand in { passes:[…], planPath, specPath }; if a large payload was dropped or stringified at the Workflow args seam, shrink it or embed the passes in the script.`)
}

const passes = input.passes
const planPath = input.planPath
const specPath = input.specPath
const priorCleared = input.priorCleared || 0  // passes cleared in earlier runs (resume); 0 on the first run

const REPORT_SCHEMA = {
  type: 'object',
  required: ['status', 'tasks'],
  properties: {
    status: { enum: ['DONE', 'BLOCKED', 'OFF_PATTERN'] },
    tasks: {
      type: 'array',
      items: {
        type: 'object',
        required: ['number', 'done'],
        properties: {
          number: { type: 'integer' },
          done: { type: 'boolean' },
          summary: { type: 'string' },
        },
      },
    },
    blockedBullet: { enum: [1, 2] },
    blockedDetail: { type: 'string' },
  },
}

const GATE_SCHEMA = {
  type: 'object',
  required: ['exitZero'],
  properties: {
    exitZero: { type: 'boolean' },
    tail: { type: 'string' },
  },
}

const AUDIT_SCHEMA = {
  type: 'object',
  required: ['count', 'path'],
  properties: {
    count: { type: 'integer' },
    path: { type: 'string' },
  },
}

const BLOCKED_CHECK_SCHEMA = {
  type: 'object',
  required: ['genuine'],
  properties: {
    genuine: { type: 'boolean' },
    reason: { type: 'string' },
  },
}

// Fresh-perspective strategist, run before the FINAL implementer attempt. Two
// attempts with the same framing have failed; a third identical dispatch would
// just reproduce them. The strategist supplies the only thing that can change
// the outcome — new input: a root-cause read the prior attempts missed and a
// concrete DIFFERENT approach, or the verdict that genuinely-new external input
// is required (→ decision-ready work-stuck).
const STRATEGIST_SCHEMA = {
  type: 'object',
  required: ['approach'],
  properties: {
    rootCause: { type: 'string' },
    approach: { type: 'string' },
    needsHuman: { type: 'boolean' },
    humanAsk: { type: 'string' },
  },
}

// Pass-start "prepare" step: a non-destructive tree snapshot (the work-absent
// baseline for this pass) folded together with the pre-flight gate run, so the
// common case spends ONE agent here instead of two. Both are mechanical
// "run a command, report" steps with no judgment — folding them does not touch
// the implementer-vs-gate-judge separation (classify stays a separate role).
// The snapshot is only consumed if a post-work flip-validation later needs to
// reconstruct the work-absent state; it is cheap to carry and never reverts the
// live tree.
const PREPARE_SCHEMA = {
  type: 'object',
  required: ['ref'],
  properties: {
    ref: { type: 'string' },
    ranGate: { type: 'boolean' },
    exitZero: { type: 'boolean' },
    tail: { type: 'string' },
    note: { type: 'string' },
  },
}

// Pre-flight classification of a gate run against the work-absent tree.
const CLASSIFY_SCHEMA = {
  type: 'object',
  required: ['kind'],
  properties: {
    kind: { enum: ['green', 'infra_red', 'right_reason_red'] },
    note: { type: 'string' },
  },
}

// Narrow existence check: is the pass's deliverable actually in the tree?
const PROBE_SCHEMA = {
  type: 'object',
  required: ['present'],
  properties: { present: { type: 'boolean' }, note: { type: 'string' } },
}

// Work-wrong vs gate-defective, when the deliverable is present but the gate red.
const DIAGNOSIS_SCHEMA = {
  type: 'object',
  required: ['verdict'],
  properties: {
    verdict: { enum: ['work_wrong', 'gate_defective'] },
    mechanism: { type: 'string' },
  },
}

// A proposed corrected gate command.
const REPAIR_SCHEMA = {
  type: 'object',
  required: ['command'],
  properties: { command: { type: 'string' }, rationale: { type: 'string' } },
}

// Whether a proposed gate flips: red against work-absent, green against work-present.
const FLIP_SCHEMA = {
  type: 'object',
  required: ['flips'],
  properties: {
    flips: { type: 'boolean' },
    redOk: { type: 'boolean' },
    greenOk: { type: 'boolean' },
    note: { type: 'string' },
  },
}

function implementerPrompt(p, idx, total, priorFailure, gateCommand) {
  const hasSpec = specPath && specPath !== 'none'
  const hasGate = gateCommand && gateCommand !== 'none' && p.endState !== 'broken-intentional'
  const redNote = p.isRedGate
    ? `\nThis is a PROOF-FIRST RED task. Its verification is \`${gateCommand}\` — it passes only when the named test FAILS. Your job is to ADD that test so it drives the real system, asserts an observable outcome, and FAILS against the current code; then leave it failing. Do NOT implement the behavior that makes it pass — a later pass does that. If your test passes on its first run it is not coupled to the missing behavior; fix the test until it fails for the right reason.\n`
    : ''
  const provesNote = p.proves
    ? `\nWhat this pass must prove (the gate's intent — build the work so a valid gate genuinely passes): ${p.proves}\n`
    : ''
  const retry = priorFailure
    ? `\n## Note from the orchestrator (prior dispatch did not clear)\n${priorFailure}\nThe working tree already holds the staged work from the prior dispatch(es) on this pass. Inspect the current state first and fix forward — repair or re-run what failed, but do NOT blindly redo a task that already landed (a non-idempotent edit applied twice corrupts the tree). Figure it out from spec intent and deliver this pass.\n`
    : ''
  return `You are the implementer for one pass of a plan, in a fresh context. The user designed this plan and trusts you to deliver this pass; the spec is the record of intent and the plan is the route — when wording is imperfect, build what the spec wants.

## Plan
Path: ${planPath}
## Spec (source of intent)
${hasSpec ? 'Path: ' + specPath : 'No separate spec file — the plan is the record of intent.'}

## This pass: Pass ${p.number} — ${p.name}
Goal: ${p.goal}
End state: ${p.endState}${p.restorer ? ' (restored by Pass ' + p.restorer + ')' : ''}
Verification: ${gateCommand || 'none'}
Position: pass ${idx + 1} of ${total}.
${provesNote}${redNote}${retry}
## Tasks in this pass (verbatim)
${(p.tasks || []).map(t => `### Task ${t.number}\n${t.text}`).join('\n\n')}

## Discipline
- Read files before editing — the tree may already hold work from earlier tasks or a prior dispatch of this pass.
- Deliver THIS pass only; do not do work that belongs to other passes.
- Default to rigor on any tradeoff the plan/spec leaves open (correctness, completeness, durability, atomicity, no-data-loss over the cheaper shape); leave a code comment naming the property you protected.
- **Completeness is the floor; the only divergence is overshoot.** Deliver the full intent of this pass's tasks and the spec stories they serve. When something is ambiguous or under-specified, resolve it by the necessity test: build whatever is *necessary* for the relevant user-outcome to actually hold — including pieces the spec never spelled out (the proto wiring, the emit site, the registration that does real work) — and nothing *adjacent* the stories do not require. NEVER defer, narrow, stub, no-op, or leave a "TODO / out of scope / later pass" in place of real work; a deferral is not a legal outcome. If you genuinely cannot finish a piece, that is a BLOCKED (below), not a silent drop.
- Checkpoint after every task with \`git add -A\` (staging, not committing). NEVER revert/reset/stash/clean the tree, even one file — fix forward by editing. Do NOT commit.
- ${p.endState === 'broken-intentional'
      ? 'This pass deliberately leaves the tree non-working; run only the per-task checks, not the whole suite.'
      : !hasGate
        ? 'This pass has no automated Verification command; run any per-task checks and report DONE when every task is complete.'
        : p.isRedGate
          ? 'After your tasks, run the Verification command and confirm it exits 0 (the new test correctly FAILS) before reporting DONE.'
          : 'After your tasks, run the Verification command and confirm it exits 0 before reporting DONE; if it fails, debug and fix — that is part of this pass. (Do NOT weaken or rewrite the Verification command to make it pass; if you believe the command itself is wrong, say so in your summary — a separate disinterested step repairs gates, not you.)'}

## Design-doc mutations (if any task here touches them)
If a task mutates a file under \`.ok-planner/design/concepts/\` or \`.ok-planner/design/tensions/\`, keep the concept/tension body self-contained: no file paths, no \`code:\`/\`pkg:\` citations, no external-doc references (\`docs/...\`, READMEs, sibling-repo paths), no quoted code or lint allowlists, no "Owns / Does NOT own" sections naming code paths. Allowed citations: other concept slugs, annotation IDs (\`@blessed-invariant: N\`), spec slugs in dated Notes entries, and dates. If a task's wording leaks a path into a new concept body, rewrite it path-free as you implement — that counts as a divergence the audit will record.

## BLOCKED — the only two reasons to stop short
1. Credentials/access only the user can provide. 2. An unauthorized destructive/irreversible action. Anything else ("needs discussion", "the plan was wrong", "more coupled than expected", "I'd like approval") is NOT blocked — make the call from spec intent and deliver. A verification command that looks wrong is NOT a block either — deliver the work; the engine repairs defective gates.

## Your return value (structured)
Return the report object: status DONE when every task is done (and, for a working/red pass, the Verification command produced the required exit code); BLOCKED with the bullet + detail only if one genuinely applies; OFF_PATTERN if you could not deliver and it is not a valid BLOCKED. List each task's number and done-state.`
}

function gatePrompt(verification) {
  return `Run this command exactly, from the repo root, and report whether it exited 0. Do not modify any files; this is a read-only gate.

    ${verification}

Return { exitZero: <true if the command's exit code was 0, else false>, tail: <last ~30 lines of output> }. Note: a command beginning with \`!\` is a red gate — it exits 0 only when the inner test FAILS, which is the intended outcome for that pass. Report the exit code as-is; do not interpret.`
}

function preparePrompt(p, gateCommand, runGate) {
  return `Prepare Pass ${p.number} with two NON-DESTRUCTIVE steps, then report. Change nothing on disk beyond the staging step 1 performs.

1. Snapshot the current working tree (the work-absent baseline for this pass — the state BEFORE Pass ${p.number}'s work) so a later step can reconstruct it without disturbing the live tree. From the repo root, run:

    git add -A && git write-tree

\`git write-tree\` records the index as a tree object and prints its SHA WITHOUT committing, stashing, moving HEAD, or changing any file. Do NOT commit, reset, checkout, or clean. Report the SHA as \`ref\` (or "" if it fails — the run continues with degraded flip-validation).

${runGate
  ? `2. Run this verification gate exactly, read-only, from the repo root, and report its exit code and last ~30 lines of output (do NOT modify any files):\n\n    ${gateCommand}\n\nSet ranGate:true, exitZero:<true iff exit 0>, tail:<output>. Note: a command beginning with \`!\` is a red gate (exit 0 means the inner test FAILED); report the exit code as-is, do not interpret.`
  : `2. Do NOT run any gate now — this pass has a proof-first red gate (exit-0-while-the-test-is-absent is its intended pre-work state) or no gate at all. Set ranGate:false.`}

Return { ref, ranGate, exitZero, tail, note }.`
}

function classifyPrompt(p, gate) {
  return `A verification gate was run on a tree where Pass ${p.number}'s work is NOT yet present. Classify its result so the engine can tell a valid gate (correctly red because the work is absent) from a defective one.

Gate exit: ${gate.exitZero ? 'ZERO (passed)' : 'NON-ZERO (failed)'}
Output tail:
${gate.tail || '(none)'}

What a passing gate should prove: ${p.proves || '(not declared)'}
Declared valid-failure signature when the work is absent (Red-when-absent): ${p.redWhenAbsent || '(not declared)'}

Classify into exactly one:
- "green" — the gate exited 0. With the work absent, a gate that already passes proves nothing (it cannot be coupled to the missing work). Choose this whenever the exit was zero.
- "infra_red" — non-zero, but the failure is the command/environment failing to RUN its check, not the thing-under-test being absent: connection refused, command not found (127), permission/exec error (126), no such file/host/service, a shell/parse/syntax error, an image build failure, a wrong subcommand (e.g. a client invoked where a server never started). The command cannot express its check here.
- "right_reason_red" — non-zero, and the failure is the thing-under-test genuinely being absent: a test assertion failed, a missing table/relation/column, an undefined symbol/route, an expected value absent. If Red-when-absent was declared, the failure matches that signature.

When the failure is genuinely ambiguous, prefer "right_reason_red" — proceeding to implement is recoverable (the post-work checks catch a residual defect), whereas a wrong "infra_red" triggers a needless pre-flight repair.

Return { kind, note: <one line: the signature you saw> }.`
}

function probePrompt(p) {
  return `The implementer reported Pass ${p.number} DONE, but its verification gate is still failing. Before deciding whether the WORK is wrong or the GATE is broken, answer one narrow question: does the pass's deliverable actually EXIST in the working tree right now?

Pass deliverable (what a passing gate would prove): ${p.proves || p.goal}

This is an EXISTENCE check, not a correctness check. Inspect the tree DIRECTLY — read the file, query the DB for the table/column, grep for the registered route or defined symbol. Do NOT run the verification command itself; that command is the thing under suspicion.

Return { present: <true if the deliverable exists, false if genuinely absent>, note: <one line: what you checked and found> }. When you genuinely cannot tell, return present: true — a false "present" is recoverable (the next step re-examines the gate), a false "absent" wrongly blames the implementer and reproduces the very failure this check exists to prevent.`
}

function diagnosisPrompt(p, gateCommand, gate) {
  return `Pass ${p.number}'s deliverable appears to EXIST in the tree, yet its verification gate is failing. Decide, skeptically, whether the WORK is wrong or the GATE command is defective.

Pass deliverable: ${p.proves || p.goal}
Declared valid-failure signature (Red-when-absent): ${p.redWhenAbsent || '(not declared)'}
Gate command:
    ${gateCommand}
Gate output tail:
${gate.tail || '(none)'}

Default to "work_wrong" — an agent under pressure will call a gate "broken" to escape a hard task, and most red gates over present-looking work mean the work is subtly incomplete. Return "gate_defective" ONLY if there is a CONCRETE mechanism by which the command cannot pass even against correct work: it invokes the wrong subcommand (e.g. \`docker compose run <svc> <client>\` starts no server), references a path/host/service that does not exist, exits 0 (or non-zero) regardless of the work, fails on setup it should have performed, or its failure is an environment/command error (connection refused, command not found) unrelated to the deliverable.

Return { verdict, mechanism: <if gate_defective, the concrete mechanism in one line; if work_wrong, what about the work looks unfinished> }.`
}

function repairPrompt(p, brokenCommand, mechanism, mode) {
  return `A verification gate for Pass ${p.number} is defective and must be repaired — WITHOUT weakening what it proves.

What the gate must prove (preserve this, or strengthen it; never weaken it): ${p.proves || p.goal}
Valid-failure signature it must exhibit when the work is absent (Red-when-absent): ${p.redWhenAbsent || '(infer a right-reason failure: the deliverable being genuinely absent)'}
The defective command:
    ${brokenCommand}
Why it is defective: ${mechanism}

Produce a corrected command that:
- proves the SAME deliverable (or strictly more) — never a weaker check that would pass without the work;
- actually runs in this environment — fix the mechanism (e.g. \`docker compose run\`→\`exec\` into the running service; bring up prerequisites first; use \`-v ON_ERROR_STOP=1\` / strict mode so a missing object is a non-zero exit, not a silent pass; name the specific test, never a blanket suite any green test satisfies);
- exits 0 ONLY when the deliverable is genuinely present, and non-zero (matching the Red-when-absent signature) when it is absent.

A correction that merely makes the gate pass — e.g. dropping strictness so it is green even with the work missing — will be REJECTED by flip-validation. ${mode === 'red_only' ? 'The work is not present yet, so this correction will first be checked to fail for the right reason now; the green half is confirmed once the work lands.' : 'This correction will be checked to flip: red against the work-absent baseline, green against the current work-present tree.'}

Return { command, rationale: <one line> }.`
}

function flipValidatePrompt(p, command, s0ref, mode) {
  if (mode === 'red_only') {
    return `Validate a repaired verification gate at PRE-FLIGHT — Pass ${p.number}'s work is NOT yet present, so check only that the command fails for the RIGHT reason now. Do not modify any files.

Repaired command:
    ${command}

What it must prove (once the work lands): ${p.proves || p.goal}
Right-reason failure signature: ${p.redWhenAbsent || '(the deliverable being genuinely absent — a failed assertion / missing object / undefined symbol, NOT a connection or command error)'}

Run the command against the current tree. It passes this check only if it exits NON-ZERO with the right-reason signature (the deliverable is genuinely absent) — NOT a connection/command/environment error, and NOT exit 0.

Return { flips: <true if it is right-reason red now>, redOk: <same>, greenOk: true, note: <one line> }.`
  }
  return `Validate that a repaired verification gate actually FLIPS around Pass ${p.number}'s work — the only proof it is coupled to the work and not a rubber stamp. Do NOT modify or revert the live working tree.

Repaired command:
    ${command}
What it must prove: ${p.proves || p.goal}
Right-reason failure signature when the work is absent: ${p.redWhenAbsent || '(the deliverable being genuinely absent)'}
Snapshot of the tree BEFORE this pass's work (work-absent baseline): tree SHA ${s0ref || '(unavailable — reconstruct another way or report you cannot)'}

Run TWO checks:
1. WORK-PRESENT (green): run the command against the current live tree/environment (this pass's work is present). It must exit 0.
2. WORK-ABSENT (red): reconstruct the work-absent state from the baseline and run the command there — it must exit NON-ZERO with the right-reason signature. Reconstruct WITHOUT touching the live tree:
   - Code/test gates: create a throwaway git worktree at the baseline tree (e.g. \`git worktree add --detach <tmp> ${s0ref || '<SHA>'}\`), run the command there, then \`git worktree remove\` it. Never revert the live tree.
   - DB/stateful gates: run against a scratch database built from the baseline (apply only the setup present in the baseline — i.e. everything EXCEPT this pass's change), or wrap a read in \`BEGIN; <drop just this pass's objects>; <run>; ROLLBACK;\`. Never mutate shared state durably.

The gate FLIPS only if check 1 is green AND check 2 is right-reason red. If you cannot faithfully reconstruct the work-absent state, set flips=false and explain — do not guess.

Return { flips, redOk: <check 2 right-reason red>, greenOk: <check 1 green>, note: <one line> }.`
}

function auditPrompt(gateCorrections) {
  const specRead = specPath && specPath !== 'none' ? `spec ${specPath}; ` : ''
  const corr = (gateCorrections && gateCorrections.length)
    ? `\n\nThe engine autonomously repaired ${gateCorrections.length} defective verification gate(s) during the run. Record EACH as a divergence (the plan authored a gate that could not pass as written — the user should see it): ${gateCorrections.map(c => `Pass ${c.pass} [${c.when}]: \`${c.from}\` → \`${c.to}\` (because ${c.why})`).join('; ')}.`
    : ''
  return `You are the divergence auditor. The implementer just finished a plan. Produce an honest record of where the working tree differs from what the plan literally said — including consequential choices the plan left unspecified (a load-bearing property traded for a cheaper shape is the most important kind). This is a record, not a critique; propose no fixes.

Read: plan ${planPath}; ${specRead}the diff (\`git diff\`, \`git diff --staged\`, \`git status\` for untracked, \`git diff --submodule=diff\` if any); and modified/new files where the diff lacks context.${corr}

CRITICAL — scan for UNDERSHOOT, which is a completion failure, not a divergence. Search the diff and the touched files for any sign that promised work was dropped rather than delivered: a stub or no-op standing in for a real outcome; a handler / endpoint / class registered or declared but doing nothing; an error class or event declared but never emitted; a config flag or field accepted but ignored; or any \`TODO\` / \`FIXME\` / "out of scope" / "deferred" / "not implemented" / "later pass" / "for now" marker on a path a spec story depends on. List each in a dedicated "## Undershoot — completion failures (must not ship)" section at the TOP of the report, with file:line. These are the exact failures the contract forbids: the spec's user-outcome stories are the floor, so an undershoot means a promised outcome is not actually delivered. (Overshoot — building unstated-but-necessary work to satisfy a story — is legal; record it in the normal divergence list, not here.)

Write the report to ${planPath.replace(/\.md$/, '')}-divergences.md as a markdown list: for each meaningful divergence, "What the plan said", "What was implemented" (file:line where useful), "Inferred reason". Include the gate corrections above as their own entries. If the implementation matches throughout and no gates were repaired, write a one-line "no meaningful divergences" report.

Return { count: <number of divergences recorded, 0 if none>, path: <the report path you wrote> }.`
}

function blockedCheckPrompt(p, report) {
  return `An implementer reported BLOCKED on Pass ${p.number} — ${p.name}. Decide, skeptically, whether the stated reason genuinely matches one of the only two permitted BLOCKED conditions:

1. Credentials or access only the user can provide (a secret, an API key, a remote resource) with no workaround.
2. An unauthorized destructive or irreversible action the plan does not authorize.

The implementer cited bullet ${report.blockedBullet || '(none given)'}: "${report.blockedDetail || ''}"

Agents under pressure mislabel coupling, ambiguity, "the plan was wrong about X", "this needs discussion", "more coupled than expected", "the verification command looks wrong", or "I'd like approval" as BLOCKED. None of those qualify — they are deliver-from-spec-intent situations (and a wrong gate is the engine's to repair, not a block). Default to NOT genuine unless the described situation plainly fits bullet 1 or 2.

Return { genuine: <true ONLY if it plainly fits bullet 1 or 2>, reason: <one line: which bullet it fits, or why it does not> }.`
}

function strategistPrompt(p, gateCommand, priorFailure) {
  const hasSpec = specPath && specPath !== 'none'
  return `Two implementer attempts on Pass ${p.number} — ${p.name} have failed the same way. A third attempt with the same framing would just reproduce the failure — your job is to supply a genuinely DIFFERENT angle, from a fresh perspective. You are not the prior implementer; do not assume its approach was the only one.

## Plan
Path: ${planPath}
${hasSpec ? '## Spec (intent)\nPath: ' + specPath + '\n' : ''}## This pass
Goal: ${p.goal}
${p.proves ? 'Must prove: ' + p.proves + '\n' : ''}Verification gate: ${gateCommand}

## What the prior attempts hit
${priorFailure}

Read the pass, the spec, and the CURRENT working tree (the prior attempts' staged work is there). Diagnose the ROOT CAUSE the prior attempts kept missing — the underlying reason they failed, not the surface symptom. Then prescribe a concrete, DIFFERENT approach for the next implementer: specifically what to do differently, not "try harder" and not a restatement of the plan.

If the real obstacle is genuinely-new external input only the user can provide — a credential, an access grant, a missing resource, or a decision the spec does not settle and you cannot settle from intent — set needsHuman:true and state exactly what to ask for. Do NOT set needsHuman just because the work is hard or coupled; that is a deliver-from-intent situation, not a blocker.

Return { rootCause, approach, needsHuman, humanAsk }.`
}

// Repair a defective gate and validate the repair by the flip test. Disinterested
// throughout (no stake in advancing the pass). Returns the adopted corrected
// command, or null if no proposal flipped within the cap.
//   mode 'red_only' — pre-flight: work is absent now; validate right-reason red now
//                     (the green half is confirmed naturally by the post-work gate).
//   mode 'full'     — post-work: work is present; validate the full flip against the
//                     work-absent baseline snapshot s0ref.
async function repairAndValidate(p, brokenCommand, mechanism, s0ref, mode) {
  let current = brokenCommand
  let why = mechanism
  for (let r = 1; r <= 2; r++) {
    const proposal = await agent(repairPrompt(p, current, why, mode), {
      label: `pass-${p.number}:repair#${r}`, phase: 'Execute', schema: REPAIR_SCHEMA,
    })
    if (!proposal || !proposal.command) {
      why = `${why}; the prior repair returned no command`
      continue
    }
    const flip = await agent(flipValidatePrompt(p, proposal.command, s0ref, mode), {
      label: `pass-${p.number}:flip#${r}`, phase: 'Execute', schema: FLIP_SCHEMA,
    })
    if (flip && flip.flips) return proposal.command
    current = proposal.command
    why = `${why}; prior repair did not flip (${(flip && flip.note) || 'no result'})`
    log(`pass ${p.number}: repair attempt ${r} did not flip, retrying`)
  }
  return null
}

phase('Execute')

// Defensive re-validation: the skill validates the passes in step 2, but the
// engine does not trust unvalidated args. It re-checks only the two invariants
// that guarantee the tree ends working — final pass `working`, every restorer
// present — and returns an `invalid-plan` escalation rather than run an
// inconsistent plan. A `working` pass with no verification is NOT rejected: the
// loop just skips its gate (the legacy single-pass fallback produces exactly
// that), and write-plan owns verification quality.
for (let i = 0; i < passes.length; i++) {
  const p = passes[i]
  if (i === passes.length - 1 && p.endState !== 'working') {
    return { status: 'escalate', kind: 'invalid-plan', pass: p.number,
             detail: `The final pass must be \`working\` (the tree must end in a working state), but Pass ${p.number} is ${p.endState}.` }
  }
  if (p.endState === 'broken-intentional' && !passes.some(q => Number(q.number) === Number(p.restorer))) {
    return { status: 'escalate', kind: 'invalid-plan', pass: p.number,
             detail: `Pass ${p.number} is broken-intentional but names restorer Pass ${p.restorer}, which is not in the plan — the tree would never be restored.` }
  }
}

const cleared = []
const gateCorrections = []  // adopted gate repairs, for the divergence log
for (let i = 0; i < passes.length; i++) {
  const p = passes[i]
  const working = p.endState !== 'broken-intentional'
  const hasGate = working && p.verification && p.verification !== 'none'
  let gateCommand = p.verification  // may be replaced by an adopted correction

  // Pass-start prepare: snapshot the work-absent baseline (for any later
  // flip-validation) AND, for a normal (non-red) gate, run the pre-flight gate —
  // folded into ONE agent since both are mechanical run-and-report steps. The
  // snapshot is taken for every gated pass (red gates need it too, for their
  // post-work flip path); only the pre-flight gate RUN is skipped for a red gate
  // (`! <test>`), whose exit-0-while-absent is its intended pre-work state.
  // classify (a judgment) stays a separate role.
  let s0ref = ''
  if (hasGate) {
    const runPreGate = !p.isRedGate
    const prep = await agent(preparePrompt(p, gateCommand, runPreGate), {
      label: `pass-${p.number}:prepare`, phase: 'Execute', schema: PREPARE_SCHEMA,
    })
    s0ref = (prep && prep.ref) || ''

    // Pre-flight classification: a non-red gate must be right-reason RED before
    // the work. green ⇒ it proves nothing; infra_red ⇒ it can't run its check —
    // either way it is defective, so repair it BEFORE dispatching the implementer
    // (don't burn implementer attempts on a broken gate).
    if (runPreGate && prep && prep.ranGate) {
      const pre = { exitZero: prep.exitZero, tail: prep.tail }
      const cls = await agent(classifyPrompt(p, pre), {
        label: `pass-${p.number}:classify`, phase: 'Execute', schema: CLASSIFY_SCHEMA,
      })
      if (cls && cls.kind !== 'right_reason_red') {
        const mech = cls.kind === 'green'
          ? `the gate passes with the work absent — it proves nothing (false-positive / too weak): ${cls.note || ''}`
          : `the command cannot run its check here — an infrastructure/command error, not the deliverable being absent: ${cls.note || (pre.tail || '').split('\n').slice(-3).join(' ')}`
        log(`pass ${p.number}: defective gate at pre-flight (${cls.kind}); repairing before implementing`)
        const fixed = await repairAndValidate(p, gateCommand, mech, s0ref, 'red_only')
        if (!fixed) {
          return { status: 'escalate', kind: 'gate-undecidable', pass: p.number, cleared,
                   detail: `Pass ${p.number}'s verification could not be made into a gate that fails for the right reason when the work is absent (pre-flight ${cls.kind}). Original: ${gateCommand}` }
        }
        gateCorrections.push({ pass: p.number, from: gateCommand, to: fixed, when: 'pre-flight', why: mech })
        gateCommand = fixed
        log(`pass ${p.number}: gate corrected at pre-flight`)
      }
    }
  }

  let priorFailure = null
  let passCleared = false
  for (let attempt = 1; attempt <= 3; attempt++) {
    // No bare repeats. Before the FINAL attempt, bring in a fresh-perspective
    // strategist: two attempts with the same framing have failed, and a third
    // identical dispatch would only reproduce them. The strategist supplies the
    // one thing that can change the outcome — new input: a root-cause read the
    // prior attempts missed and a DIFFERENT approach, or the verdict that
    // genuinely-new external input is required (escalate work-stuck early, with
    // a decision-ready ask, rather than waste the last attempt).
    if (attempt === 3 && priorFailure) {
      const strat = await agent(strategistPrompt(p, gateCommand, priorFailure), {
        label: `pass-${p.number}:strategist`, phase: 'Execute', schema: STRATEGIST_SCHEMA,
      })
      if (strat && strat.needsHuman) {
        return { status: 'escalate', kind: 'work-stuck', pass: p.number,
                 lastFailure: priorFailure, rootCause: strat.rootCause || '',
                 humanAsk: strat.humanAsk || '', cleared }
      }
      if (strat && strat.approach) {
        priorFailure = `${priorFailure}\n\n## Fresh-perspective strategist (two prior attempts failed — do NOT repeat them)\nRoot cause the prior attempts missed: ${strat.rootCause || '(unstated)'}\nTake this DIFFERENT approach: ${strat.approach}`
        log(`pass ${p.number}: injected a fresh-perspective strategy before the final attempt`)
      }
    }
    const report = await agent(implementerPrompt(p, i, passes.length, priorFailure, gateCommand), {
      label: `pass-${p.number}:impl#${attempt}`,
      phase: 'Execute',
      schema: REPORT_SCHEMA,
    })
    if (!report) {
      priorFailure = 'The prior dispatch was skipped before returning a report; re-dispatching.'
      log(`pass ${p.number}: dispatch skipped on attempt ${attempt}, re-dispatching`)
      continue
    }
    if (report.status === 'BLOCKED') {
      // Don't take BLOCKED at face value — a separate skeptic re-checks it
      // against the two permitted bullets. Misapplied BLOCKEDs re-dispatch.
      const verdict = await agent(blockedCheckPrompt(p, report), {
        label: `pass-${p.number}:blocked-check#${attempt}`,
        phase: 'Execute',
        schema: BLOCKED_CHECK_SCHEMA,
      })
      if (!verdict || verdict.genuine) {
        // Confirmed genuine, or the skeptic could not adjudicate → surface to the human.
        return { status: 'escalate', kind: 'blocked', pass: p.number,
                 bullet: report.blockedBullet, detail: report.blockedDetail || '',
                 cleared }
      }
      priorFailure = `You reported BLOCKED, but that is not a permitted block: ${verdict.reason}. The only valid blocks are (1) credentials/access only the user can provide and (2) an unauthorized destructive action. Make the call from spec intent and deliver this pass.`
      log(`pass ${p.number}: misapplied BLOCKED on attempt ${attempt}, re-dispatching`)
      continue
    }
    if (report.status !== 'DONE') {
      priorFailure = `Prior dispatch returned ${report.status} — not a clean DONE and not a valid BLOCKED.`
      log(`pass ${p.number}: off-pattern on attempt ${attempt}, re-dispatching`)
      continue
    }
    // DONE claimed → confirm every task in this pass is actually marked done.
    const doneNumbers = new Set((report.tasks || []).filter(t => t.done).map(t => Number(t.number)))
    const missing = (p.tasks || []).filter(t => !doneNumbers.has(Number(t.number))).map(t => t.number)
    if (missing.length) {
      priorFailure = `Report claimed DONE, but task(s) ${missing.join(', ')} in this pass are not marked done in the per-task list. Every task in the pass must be delivered.`
      log(`pass ${p.number}: DONE claimed with undone task(s) ${missing.join(', ')}, re-dispatching`)
      continue
    }
    // broken-intentional / "none" verifications skip the gate.
    if (!hasGate) {
      passCleared = true
      break
    }
    // Post-work gate.
    const gate = await agent(gatePrompt(gateCommand), {
      label: `pass-${p.number}:gate#${attempt}`,
      phase: 'Execute',
      schema: GATE_SCHEMA,
    })
    if (!gate) {
      priorFailure = 'The verification gate was skipped before returning a result; re-running the pass.'
      log(`pass ${p.number}: gate skipped on attempt ${attempt}, re-dispatching`)
      continue
    }
    if (gate.exitZero) {
      // Pre-flight established the gate was right-reason red (or it was corrected
      // to be); a green here means it FLIPPED — the proof the gate is coupled to
      // the work. Advance.
      passCleared = true
      break
    }
    // Still red after a credible DONE. Is the deliverable even present?
    const probe = await agent(probePrompt(p), {
      label: `pass-${p.number}:probe#${attempt}`, phase: 'Execute', schema: PROBE_SCHEMA,
    })
    const tail = (gate.tail || '').split('\n').slice(-30).join('\n    ')
    if (probe && probe.present === false) {
      priorFailure = `Verification still fails and this pass's deliverable is not yet present in the tree (${probe.note || ''}):\n\n    $ ${gateCommand}\n    ${tail}\n\nThe work is incomplete — finish it.`
      log(`pass ${p.number}: gate red, deliverable absent on attempt ${attempt}, re-dispatching`)
      continue
    }
    // Deliverable present but gate red → diagnose work-wrong vs gate-defective.
    const diag = await agent(diagnosisPrompt(p, gateCommand, gate), {
      label: `pass-${p.number}:diagnose#${attempt}`, phase: 'Execute', schema: DIAGNOSIS_SCHEMA,
    })
    if (!diag || diag.verdict === 'work_wrong') {
      priorFailure = `Verification still fails though the deliverable is present (${(diag && diag.mechanism) || 'work appears unfinished'}):\n\n    $ ${gateCommand}\n    ${tail}\n\nDebug and fix the work. Do not touch the verification command.`
      log(`pass ${p.number}: gate red, diagnosed work-wrong on attempt ${attempt}, re-dispatching`)
      continue
    }
    // gate-defective → repair, validated by the full flip (red@baseline, green@live).
    log(`pass ${p.number}: gate diagnosed defective post-work; repairing`)
    const fixed = await repairAndValidate(p, gateCommand, diag.mechanism, s0ref, 'full')
    if (!fixed) {
      return { status: 'escalate', kind: 'gate-undecidable', pass: p.number, cleared,
               detail: `Pass ${p.number}'s deliverable is present but the verification is defective and no repair produced a gate that flips (red without the work, green with it). Mechanism: ${diag.mechanism}. Original: ${gateCommand}` }
    }
    gateCorrections.push({ pass: p.number, from: gateCommand, to: fixed, when: 'post-work', why: diag.mechanism })
    gateCommand = fixed
    // The repaired gate flipped: green against the live (work-present) tree and
    // red against the baseline — the pass is genuinely done by a valid gate.
    passCleared = true
    log(`pass ${p.number}: gate corrected post-work and flipped; pass cleared`)
    break
  }
  if (!passCleared) {
    // Decision-ready: two attempts PLUS a fresh-perspective strategist-directed
    // attempt all failed this valid gate, so it genuinely needs new input from a
    // human — `priorFailure` carries the strategist's root-cause read and the
    // different approach that was tried and still failed.
    return { status: 'escalate', kind: 'work-stuck', pass: p.number,
             lastFailure: priorFailure,
             note: 'Two attempts plus a fresh-perspective strategist-directed attempt all failed this gate (which is valid — a defective gate would have been repaired, not counted here). It needs new input from you; lastFailure carries the root-cause read and the different approach already tried.',
             cleared }
  }
  cleared.push(p.number)
  log(`pass ${p.number} cleared (${cleared.length}/${passes.length})`)
}

phase('Audit')
const audit = await agent(auditPrompt(gateCorrections), { label: 'divergence-audit', phase: 'Audit', schema: AUDIT_SCHEMA })

return { status: 'complete', passesCleared: priorCleared + cleared.length, clearedThisRun: cleared.length, gateCorrections, divergences: audit || { count: 0, path: 'none' } }
```

## Rules

- One implementer at a time per pass (the loop is sequential — passes
  depend on prior passes). The workflow never runs passes in parallel.
- The gate branch is code: a `working` pass advances **only** when the
  separate gate agent reports `exitZero: true` AND that green is a *flip*
  (the gate was right-reason red before the work, or was corrected to a
  gate that flips). The exit code is still read by an agent — a workflow
  script can't run a shell — but the "close enough" discretion that was
  the prose loop's failure mode is gone.
- **Defective gates are repaired, not escalated.** A gate that is green
  with the work absent (proves nothing) or fails with an
  infrastructure/command error (can't run its check) is repaired by a
  disinterested agent and adopted **only if the repair itself flips**
  (red against the work-absent baseline, green against the work-present
  tree). The implementer never edits a gate; the only corrections that
  survive keep their discriminating power, so a gate cannot be weakened
  to pass. Every adopted correction is logged in `gateCorrections` and
  recorded as a divergence.
- A red gate (`! <test>`) is a normal `working` gate that skips pre-flight
  repair: `exitZero: true` means the new test correctly failed before its
  fix lands. The implementer for a red pass must not make the test pass.
- The work-absent state for flip-validation is reconstructed
  non-destructively — a throwaway `git worktree` at the pre-pass tree
  snapshot for code/test gates, a scratch DB or `BEGIN…ROLLBACK` for
  stateful gates. Never revert the live tree.
- Never commit; never revert/reset/stash/clean (binds the skill and
  every agent). Implementers and the pass-start prepare step use
  `git add -A` / `git write-tree` (non-destructive); recovery is
  fix-forward.
- Escalations are the only return-to-human boundaries, and each names a
  real blocker: `blocked` (skeptic-confirmed credentials/destruction),
  `work-stuck` (a *valid, flipping* gate the work can't satisfy after the
  implementer cap), `gate-undecidable` (a defective gate no repair could
  make flip), `invalid-plan` (structural). A "trivially fixable gate
  command error" is **not** an escalation — the engine repairs it and
  records it as a divergence. Everything the loop can re-dispatch
  (off-pattern, work-wrong red, a DONE that left tasks undone, misapplied
  BLOCKED) it re-dispatches itself, up to the cap.
- No bare repeats. A re-dispatch always carries new input, or it would
  just reproduce the prior result. Attempt 2 carries the specific failure;
  before the final attempt a fresh-perspective **strategist** supplies a
  root-cause read and a *different* approach (or the verdict that
  genuinely-new external input is required). The engine escalates
  `work-stuck` precisely when this perspective ladder is exhausted — i.e.
  when a human is the only remaining source of new input — and the
  escalation is decision-ready (root cause + the different approach
  already tried). Raising the cap without adding perspective is not a
  fix; the strategist is.
