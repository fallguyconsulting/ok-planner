---
name: execute-plan-via-workflow
description: "ONLY activated by explicit /execute-plan-via-workflow slash command. Experimental workflow-backed variant of execute-plan: the deterministic pass loop (dispatch → judge → gate → re-dispatch → converge → divergence audit) runs as a Workflow with code gates; human-facing intake, escalation, final review, and the divergence walk stay in the skill. Never auto-triggered."
---

# Executing a Plan via a Workflow engine (experimental)

Same contract as `execute-plan` — drive a plan to completion pass by
pass, then audit, review, and walk divergences — but the **control flow
is code, not prose**. The dispatch / judge / gate / re-dispatch /
converge loop runs inside a `Workflow` whose gates are real branches on
structured agent output, so a pass cannot advance on a verification the
orchestrator "decided" was close enough. The prose-orchestrated
`execute-plan` remains the default; this is the variant under trial.

**Why this exists.** A prose orchestrator can skip a gate, rationalize a
DONE, mis-count re-dispatches, or advance on a green-but-shallow test.
The workflow's control flow can't: `if (!gate.exitZero) redispatch()` is
code, and the exit code it branches on is read by a **separate** gate
agent whose only job is to report the boolean — so the agent that wants
the pass to advance is no longer the one judging whether it may. That
does not make the gate fully mechanical (an LLM still reports the exit
code; a workflow script can't run a shell), but it removes the "close
enough" discretion that was the prose loop's failure mode. The
proof-first red/green discipline from `write-plan` rides along: a red
task's `! <test>` verification gates exactly as written, and a shape
test that is green-from-birth makes `! <test>` exit non-zero, fails the
gate, and re-dispatches.

## What runs where (the seam)

| In the **workflow** (code, headless) | In the **skill** (prose, human-facing) |
|---|---|
| dispatch one implementer agent per pass | parse the plan → structured passes |
| judge the report (schema): every task done, BLOCKED re-checked by a skeptic | the escalation conversation (genuine BLOCKED, cap reached, invalid plan) |
| run the verification gate, advance or re-dispatch (cap 3) | the final review (`review-work` → `review-cleanup`) |
| converge all passes, then the divergence audit | the divergence walk + rework decisions, archive |

v1 deliberately keeps the **final review** skill-side (reuse
`review-work`/`review-cleanup` as-is) and keeps **resume across an
escalation** skill-side (a fixed BLOCKED is resumed by re-invoking the
workflow on the remaining passes). Moving the review-to-clean loop into
the workflow is a natural v2; so is multi-session durability (which is
what rimsky is meant to provide). Note them; don't build them here.

## Opt-in

Invoking the `Workflow` tool from this skill is authorized: the user ran
`/execute-plan-via-workflow`, and this skill's instructions tell you to
call it. That is a valid Workflow opt-in. Pass the structured passes as
`args`; do not paste the plan text into the script.

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
     tasks: [ { number, text } ] }              // task text VERBATIM from the plan
   ```
   If the plan has no passes at all (an older flat-task plan),
   synthesize a single pass — `number: 1`, `name` from the plan title,
   `endState: "working"`, the plan's overall verification command (or
   `none` if it has none), `isRedGate` set accordingly, and every task —
   and tell the user in one line before running that the plan predates
   the pass model and is being executed as a single pass (same fallback
   as `execute-plan` step 1).

   Validate before running: every `broken-intentional` pass names a
   restorer that exists; the final pass is `working`; `working` passes
   have a non-empty verification (the exception is the synthesized legacy
   single pass above, which may be `none`), `broken-intentional` passes
   have `none`. If the plan is internally inconsistent, **stop and
   surface to the user** — that is a planner defect (same as
   `execute-plan` step 3d), not something the workflow should run into.
   (The engine independently re-checks the two invariants that guarantee
   the tree ends working — final pass `working`, every restorer present —
   and returns an `invalid-plan` escalation if either is violated, see
   step 5; catching the rest here keeps it out of a workflow run.)

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
     user, then go to step 6.
   - `status: "escalate"` → the workflow stopped at a boundary only a
     human can clear. Surface it verbatim and **wait**:
     - `kind: "blocked"` — a BLOCKED the engine's skeptic confirmed
       genuine (bullet 1 credentials/access, or bullet 2 unauthorized
       destructive action) on `pass`, or one it could not adjudicate.
       Misapplied BLOCKEDs never reach you — the engine re-dispatches
       those itself. Give the user the bullet, the detail, and the
       passes already cleared.
     - `kind: "cap-reached"` — 3 dispatches on `pass` without a clean
       DONE+gate. Surface the last report and the options: sharpen the
       brief and resume, rework the pass/plan, or abandon.
     - `kind: "invalid-plan"` — the engine's defensive validation found
       a passes structure that can't end in a working tree (final pass
       not `working`, or a `broken-intentional` pass naming a restorer
       that isn't in the plan). This should have been caught in step 2;
       surface the `detail` and the offending `pass` and take it back to
       `write-plan` (or fix the plan inline). Do not resume until the
       plan is fixed.
     When the user resolves a `blocked` or `cap-reached`, **resume** by
     re-invoking the workflow (step 4) with `args.passes` set to the
     remaining passes (the stuck pass onward) and `args.priorCleared`
     raised by this escalation's `cleared` count (so the final
     `passesCleared` total stays accurate across runs). A fresh run from
     the stuck pass is correct because the fix changed the tree; do not
     journal-resume across a human fix.

6. **Final review (skill-side).** Invoke `ok-planner:review-work` for the
   full implementation; let `review-cleanup` drive the fix cycle to
   clean. (Reused unchanged. The workflow already ran the proof-first
   gates pass-by-pass; this is the holistic pass.)

6a. **Regenerate `concepts.md`** if any pass touched
   `.ok-planner/design/concepts/`, per `execute-plan` step 5a.

7. **Walk the divergence report with the user** — every entry, as the
   table in `execute-plan` step 6. Reworks come from this conversation.

8. **Archive** the plan, its `-divergences.md`, and the linked spec into
   `.ok-planner/history/` (per `execute-plan` step 7).

## The workflow engine

Pass this as the `Workflow` `script`. It is plan-agnostic — it reads
everything from `args`. The verification strings already encode the
proof-first semantics (a red task's command starts with `!`), so the
gate needs no special-casing beyond running the command and reading the
exit code.

```javascript
export const meta = {
  name: 'execute-plan-engine',
  description: 'Deterministic pass-by-pass execution engine for an ok-planner plan: dispatch the implementer, judge a structured report, run the verification gate as code, re-dispatch up to a cap, converge every pass, then run the divergence audit.',
  phases: [
    { title: 'Execute', detail: 'one implementer per pass, code-gated' },
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

function implementerPrompt(p, idx, total, priorFailure) {
  const hasSpec = specPath && specPath !== 'none'
  const redNote = p.isRedGate
    ? `\nThis is a PROOF-FIRST RED task. Its verification is \`${p.verification}\` — it passes only when the named test FAILS. Your job is to ADD that test so it drives the real system, asserts an observable outcome, and FAILS against the current code; then leave it failing. Do NOT implement the behavior that makes it pass — a later pass does that. If your test passes on its first run it is not coupled to the missing behavior; fix the test until it fails for the right reason.\n`
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
Verification: ${p.verification}
Position: pass ${idx + 1} of ${total}.
${redNote}${retry}
## Tasks in this pass (verbatim)
${(p.tasks || []).map(t => `### Task ${t.number}\n${t.text}`).join('\n\n')}

## Discipline
- Read files before editing — the tree may already hold work from earlier tasks or a prior dispatch of this pass.
- Deliver THIS pass only; do not do work that belongs to other passes.
- Default to rigor on any tradeoff the plan/spec leaves open (correctness, completeness, durability, atomicity, no-data-loss over the cheaper shape); leave a code comment naming the property you protected.
- Checkpoint after every task with \`git add -A\` (staging, not committing). NEVER revert/reset/stash/clean the tree, even one file — fix forward by editing. Do NOT commit.
- ${p.endState === 'broken-intentional'
      ? 'This pass deliberately leaves the tree non-working; run only the per-task checks, not the whole suite.'
      : (!p.verification || p.verification === 'none')
        ? 'This pass has no automated Verification command; run any per-task checks and report DONE when every task is complete.'
        : p.isRedGate
          ? 'After your tasks, run the Verification command and confirm it exits 0 (the new test correctly FAILS) before reporting DONE.'
          : 'After your tasks, run the Verification command and confirm it exits 0 before reporting DONE; if it fails, debug and fix — that is part of this pass.'}

## Design-doc mutations (if any task here touches them)
If a task mutates a file under \`.ok-planner/design/concepts/\` or \`.ok-planner/design/tensions/\`, keep the concept/tension body self-contained: no file paths, no \`code:\`/\`pkg:\` citations, no external-doc references (\`docs/...\`, READMEs, sibling-repo paths), no quoted code or lint allowlists, no "Owns / Does NOT own" sections naming code paths. Allowed citations: other concept slugs, annotation IDs (\`@blessed-invariant: N\`), spec slugs in dated Notes entries, and dates. If a task's wording leaks a path into a new concept body, rewrite it path-free as you implement — that counts as a divergence the audit will record.

## BLOCKED — the only two reasons to stop short
1. Credentials/access only the user can provide. 2. An unauthorized destructive/irreversible action. Anything else ("needs discussion", "the plan was wrong", "more coupled than expected", "I'd like approval") is NOT blocked — make the call from spec intent and deliver.

## Your return value (structured)
Return the report object: status DONE when every task is done (and, for a working/red pass, the Verification command produced the required exit code); BLOCKED with the bullet + detail only if one genuinely applies; OFF_PATTERN if you could not deliver and it is not a valid BLOCKED. List each task's number and done-state.`
}

function gatePrompt(verification) {
  return `Run this command exactly, from the repo root, and report whether it exited 0. Do not modify any files; this is a read-only gate.

    ${verification}

Return { exitZero: <true if the command's exit code was 0, else false>, tail: <last ~30 lines of output> }. Note: a command beginning with \`!\` is a red gate — it exits 0 only when the inner test FAILS, which is the intended outcome for that pass. Report the exit code as-is; do not interpret.`
}

function auditPrompt() {
  const specRead = specPath && specPath !== 'none' ? `spec ${specPath}; ` : ''
  return `You are the divergence auditor. The implementer just finished a plan. Produce an honest record of where the working tree differs from what the plan literally said — including consequential choices the plan left unspecified (a load-bearing property traded for a cheaper shape is the most important kind). This is a record, not a critique; propose no fixes.

Read: plan ${planPath}; ${specRead}the diff (\`git diff\`, \`git diff --staged\`, \`git status\` for untracked, \`git diff --submodule=diff\` if any); and modified/new files where the diff lacks context.

Write the report to ${planPath.replace(/\.md$/, '')}-divergences.md as a markdown list: for each meaningful divergence, "What the plan said", "What was implemented" (file:line where useful), "Inferred reason". If the implementation matches throughout, write a one-line "no meaningful divergences" report.

Return { count: <number of divergences recorded, 0 if none>, path: <the report path you wrote> }.`
}

function blockedCheckPrompt(p, report) {
  return `An implementer reported BLOCKED on Pass ${p.number} — ${p.name}. Decide, skeptically, whether the stated reason genuinely matches one of the only two permitted BLOCKED conditions:

1. Credentials or access only the user can provide (a secret, an API key, a remote resource) with no workaround.
2. An unauthorized destructive or irreversible action the plan does not authorize.

The implementer cited bullet ${report.blockedBullet || '(none given)'}: "${report.blockedDetail || ''}"

Agents under pressure mislabel coupling, ambiguity, "the plan was wrong about X", "this needs discussion", "more coupled than expected", or "I'd like approval" as BLOCKED. None of those qualify — they are deliver-from-spec-intent situations. Default to NOT genuine unless the described situation plainly fits bullet 1 or 2.

Return { genuine: <true ONLY if it plainly fits bullet 1 or 2>, reason: <one line: which bullet it fits, or why it does not> }.`
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
for (let i = 0; i < passes.length; i++) {
  const p = passes[i]
  let priorFailure = null
  let passCleared = false
  for (let attempt = 1; attempt <= 3; attempt++) {
    const report = await agent(implementerPrompt(p, i, passes.length, priorFailure), {
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
    // (Gate-skipped broken-intentional passes have no other check.)
    const doneNumbers = new Set((report.tasks || []).filter(t => t.done).map(t => Number(t.number)))
    const missing = (p.tasks || []).filter(t => !doneNumbers.has(Number(t.number))).map(t => t.number)
    if (missing.length) {
      priorFailure = `Report claimed DONE, but task(s) ${missing.join(', ')} in this pass are not marked done in the per-task list. Every task in the pass must be delivered.`
      log(`pass ${p.number}: DONE claimed with undone task(s) ${missing.join(', ')}, re-dispatching`)
      continue
    }
    // broken-intentional / "none" verifications skip the gate.
    if (p.endState === 'broken-intentional' || !p.verification || p.verification === 'none') {
      passCleared = true
      break
    }
    const gate = await agent(gatePrompt(p.verification), {
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
      passCleared = true
      break
    }
    priorFailure = `Pass reported DONE but the Verification command did not produce exit 0:\n\n    $ ${p.verification}\n    ${(gate.tail || '').split('\n').slice(-30).join('\n    ')}\n\nDebug and fix. The pass is not done until Verification passes (for a red gate, until the new test correctly fails).`
    log(`pass ${p.number}: gate failed on attempt ${attempt}, re-dispatching`)
  }
  if (!passCleared) {
    return { status: 'escalate', kind: 'cap-reached', pass: p.number,
             lastFailure: priorFailure, cleared }
  }
  cleared.push(p.number)
  log(`pass ${p.number} cleared (${cleared.length}/${passes.length})`)
}

phase('Audit')
const audit = await agent(auditPrompt(), { label: 'divergence-audit', phase: 'Audit', schema: AUDIT_SCHEMA })

return { status: 'complete', passesCleared: priorCleared + cleared.length, clearedThisRun: cleared.length, divergences: audit || { count: 0, path: 'none' } }
```

## Rules

- One implementer at a time per pass (the loop is sequential — passes
  depend on prior passes). The workflow never runs passes in parallel.
- The gate branch is code: a `working` pass advances **only** when the
  separate gate agent reports `exitZero: true`. The exit code is still
  read and reported by an agent — a workflow script can't run a shell —
  but the "close enough" discretion that was the prose loop's failure
  mode is gone: the gate agent only reports a boolean and is not the
  agent trying to advance the pass.
- A red gate (`! <test>`) is a normal `working` gate: `exitZero: true`
  means the new test correctly failed before its fix lands. The
  implementer for a red pass must not make the test pass.
- Never commit; never revert/reset/stash/clean (binds the skill and
  every agent). Implementers checkpoint with `git add -A`; recovery is
  fix-forward.
- Escalations are the only return-to-human boundaries. Everything the
  prose `execute-plan` would re-dispatch (off-pattern, gate failure, a
  DONE that left tasks undone, misapplied BLOCKED) the workflow
  re-dispatches itself, up to the cap. A BLOCKED escalates only after a
  separate skeptic agent confirms it fits one of the two permitted
  bullets (or cannot adjudicate it); a misapplied BLOCKED is
  re-dispatched like any other off-pattern report.
- This is experimental. If the workflow behaves worse than the prose
  `execute-plan` on a given plan, fall back to `/execute-plan` — both
  read the same plan format and the same proof-first verifications.
