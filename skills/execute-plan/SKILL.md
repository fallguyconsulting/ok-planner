---
name: execute-plan
description: "ONLY activated by explicit /execute-plan slash command or ok-planner pipeline. Never auto-triggered. Drives a plan to completion pass by pass as a Workflow with an opposing implementer ↔ validator pair: dispatch the implementer, dispatch a disinterested validator that argues against the pass delivering, loop on rejection, escalate on exhaustion. Every three cleared passes, an interim review-and-fix checkpoint sweeps the accumulated work so issues don't pile up to the end. After all passes clear, a final verification gate runs the project's build + test + lint suite; the no-deferral audit and completion auditor follow. Human-facing intake, escalation, final review, and the closing walk stay in the skill."
---

# Executing a Plan (Workflow engine)

Drive a plan to completion pass by pass with an opposing implementer
↔ validator pair, then audit and review. The control flow is code,
not prose: dispatch → validate → loop or advance is a `Workflow`
whose branches are real conditions on structured agent output, so a
pass cannot advance on a verdict the orchestrator "decided" was
close enough.

**Why an opposing pair, not mechanical gates.** Agents are smart and
effective when given the right context; what they fail at is being
their own check on whether they delivered. Splitting the work
across an implementer (which wants the pass to advance) and a
validator (whose job is to argue it didn't) gets the adversarial
discipline without the flip-coupled gate command the planner used
to have to author. The validator reads the spec, the pass's
Falsifier brief, and the diff, and argues from intent. There is no
shell command coupling required; the validator is the judge.

## The opposing pair, per pass

Two agents per pass, with one adversarial loop between them:

- **Implementer** — given the spec, the pass goal, the tasks, and
  the pass's `Falsifier:` brief, builds the work and reports
  `DONE` / `BLOCKED` / `OFF_PATTERN`. The only role that wants the
  pass to advance.
- **Validator** — given the spec, the pass goal, the falsifier
  brief, and the diff, argues from spec intent that the pass does
  NOT deliver. Disinterested. Defaults to "does not deliver";
  confirms only when it can find no evidence the falsifier holds,
  with concrete citation against the diff. Rejects with a specific
  critique the implementer can act on.

A validator-rejected DONE re-dispatches the implementer with the
validator's critique as input. Cap at **2 implementer attempts per
pass**; on the second failure, escalate as `work-stuck`.

A `BLOCKED` report triggers the **strategist** — one agent,
invoked on the *first* BLOCKED (no waiting). The strategist
combines what were previously the blocked-check skeptic and the
fresh-perspective strategist into one role. It either confirms the
BLOCKED is genuine (escalate as `blocked`) or supplies a
root-cause read and a different approach (re-dispatch with new
input). **One strategist intervention per pass**; a second BLOCKED
report after a strategist intervention escalates as `work-stuck`.

Every role in the engine except the implementer is
**disinterested** — none has a stake in advancing the pass.

## Pass model

Every pass leaves the tree in a working state — there is no
`broken-intentional` end-state and no restorer model. A pass's
contract is its **Falsifier brief**, written by `write-plan`.
Acceptance passes additionally annotate every story slug they serve
in their header (`acceptance pass — STORY-<slug>`, or a
comma-separated list for a shared pass); the validator
uses the spec's story (Acceptance + Falsifier + Proof) as its
deeper context for those passes.

## What runs where (the seam)

| In the workflow (code, headless) | In the skill (prose, human-facing) |
|---|---|
| dispatch implementer; dispatch validator; loop on rejection | parse the plan → structured passes |
| interim review-and-fix checkpoint every 3 cleared passes | escalation conversation (blocked, work-stuck, invalid-plan) |
| strategist on first BLOCKED | |
| final verification gate (build + test + lint per project docs) | the closing walk (proofs working + decisions kept + decisions diverged + coverage divergences) |
| no-deferral audit as gate (loops with fixer until clean) | final review (`review-work` → `review-cleanup`) |
| completion auditor (writes the four-section report, runs coverage + intent-drift audit) | archive |
| advance per pass; converge; verify; closing audits | |

The **final review** stays skill-side (`review-work` /
`review-cleanup`, unchanged — known to converge well). **Resume
across an escalation** stays skill-side (a fixed `blocked` is
resumed by re-invoking the workflow on the remaining passes).

## Opt-in

Invoking the `Workflow` tool from this skill is authorized: the user
ran `/execute-plan` (or the ok-planner pipeline reached it), and this
skill's instructions tell you to call it. That is a valid Workflow
opt-in. Pass the structured passes as `args`; do not paste the plan
text into the script. (For a plan too large for the inline `args`
channel — it can drop or stringify a big payload — embed the passes
in the script instead, as the engine's input guard notes.)

## Process

1. **Select the plan.** If the user named a plan, use it. Otherwise
   list `.ok-planner/plans/*.md` (newest first, filename + title line)
   and ask which to run. Read it in full. Read the `**Spec:**` header.

2. **Parse the plan into structured passes.** For each pass capture:
   ```
   { number,
     name,
     goal,
     falsifier,           // the Falsifier brief — the validator's contract
     isAcceptancePass,    // true if the pass header annotates an acceptance pass
     storySlug,           // the story slug if isAcceptancePass, else "";
                          // a shared acceptance pass lists every slug it
                          // serves, comma-separated ("a, b")
     tasks: [ { number, text } ] }   // task text VERBATIM from the plan
   ```
   If any pass has no falsifier, **stop and surface to the user** —
   that is a planner defect (`write-plan`'s reviewer should have
   caught it). The engine cannot run without a falsifier; the
   validator has nothing to argue against. Same for acceptance
   passes missing their story slug annotation.

3. **Affirm the layout.** Invoke `ok-planner:affirm`.

4. **Run the engine as a workflow.** Call `Workflow` with the script
   in "The workflow engine" below and:
   ```
   args = { passes: <the parsed pass array>,
            planPath: "<plan path>",
            specPath: "<spec path from the **Spec:** header, or 'none'>",
            priorCleared: 0 }   // raised on resume across an escalation
   ```
   The workflow runs in the background; you are re-invoked when it
   completes. Its return value is a result object — act on its
   `status`.

5. **Handle the result.**
   - `status: "complete"` → the workflow cleared every pass, the
     final verification gate passed, the no-deferral audit ran to
     clean, and the completion report was produced. Surface the
     report path and a one-line summary, then go to step 6.
   - `status: "escalate"` → the workflow stopped at a boundary only a
     human can clear. Surface it verbatim and **wait**:
     - `kind: "blocked"` — the strategist confirmed a BLOCKED is
       genuine (credentials/access only the user can provide, or an
       unauthorized destructive action). Give the user the bullet,
       the detail, the strategist's `humanAsk` if any, and the
       passes already cleared.
     - `kind: "work-stuck"` — the implementer ↔ validator pair could
       not converge on the pass after the cap, OR a second BLOCKED
       came in after a strategist intervention. `lastFailure`
       carries the validator's last critique or the strategist's
       root-cause read and different approach already tried. Surface
       with options: sharpen the brief and resume, rework the
       pass/plan, or abandon.
     - `kind: "invalid-plan"` — the engine's defensive validation
       found a pass missing its Falsifier brief, or an acceptance
       pass missing its story slug annotation. Take it back to
       `write-plan`.

     When the user resolves a `blocked` or `work-stuck`, **resume**
     by re-invoking the workflow (step 4) with `args.passes` set to
     the remaining passes (the stuck pass onward) and
     `args.priorCleared` raised by this escalation's `cleared`
     count.

6. **Final review (skill-side).** Invoke `ok-planner:review-work` for
   the full implementation; let `review-cleanup` drive the fix cycle
   to clean. The workflow's no-deferral audit already enforced
   completion; this is the holistic code-quality pass.

6a. **Regenerate design TOCs** if any pass touched
    `.ok-planner/design/concepts/`, `.ok-planner/design/stories/`, or
    `.ok-planner/design/decisions/`. For each touched directory,
    refresh its TOC (`concepts.md`, `stories.md`, or `decisions.md`)
    using the same format `discover-design`'s "Regenerate the design
    catalog summaries" step produces — read every file in the
    directory, extract slug + one-line summary, write the sorted
    alphabetical list under the standard header.

7. **Walk the completion report with the user** — four sections, in
   order, every entry:
   - **Proof walkthrough** — every story's proof artifact, exhibited
     working. Surface each demo/example/proof and walk it with the
     user. Gaps here go back to the implementer (the no-deferral
     audit should have caught them; flag a defect if any made it
     through).
   - **Technical decisions kept** — every spec TD the implementation
     honored as decided. Explicit enumeration; the user signs off.
   - **Technical decisions diverged** — every choice that changed,
     with flavor (improved / selected / necessitated) and reason.
     The user decides what (if anything) to rework.
   - **Coverage divergences** — findings from the closing coverage
     + intent-drift audit: coverage gaps (stories with no
     `@story:<slug>` annotation in the codebase), intent drifts
     (proof files no longer satisfying their story's Proof field),
     and dangling annotations (`@story:<slug>` annotations pointing
     at retired or missing stories). For each finding, the user
     adjudicates: accept as informational, course-correct now (open
     a brainstorm to address), or bounce back to the implementer
     (process defect). Ideally this section is empty — the
     brainstorm dialogue gate and the validator's per-pass checks
     should have caught everything at change points. When it isn't
     empty, the entries are the safety net catching drift the
     earlier gates missed.

   The four sections together cover 100% of the spec manifest's
   stories + TDs, plus any coverage drift the closing audit
   surfaced. The report is the durable record; what the user
   walks IS what goes to history.

8. **Archive** the plan, its `<plan>-completion-report.md`, and the
   linked spec into `.ok-planner/history/`:
   - `<plan>.md` and `<plan>-completion-report.md` →
     `.ok-planner/history/plans/`
   - spec from the plan's `**Spec:**` header →
     `.ok-planner/history/specs/`

   Use `git mv` if the project tracks `.ok-planner/` in git, otherwise
   plain `mv`. Skip the spec move if `**Spec:**` is `none`, missing,
   or points to a file that doesn't exist. Tell the user what was
   moved in one short line.

## The workflow engine

Pass this as the `Workflow` `script`. It is plan-agnostic — it reads
everything from `args`. Each pass runs the implementer ↔ validator
loop; the validator is the judge of whether the pass delivered.

```javascript
export const meta = {
  name: 'execute-plan-engine',
  description: 'Opposing implementer ↔ validator pair per pass, with a strategist on BLOCKED. Loops on validator rejection; caps at 2 implementer attempts; escalates on exhaustion. Every 3 cleared passes, an interim review-and-fix checkpoint sweeps the accumulated work. After all passes clear, the final verification gate runs the project\'s build + test + lint suite; the no-deferral audit + completion auditor follow.',
  phases: [
    { title: 'Execute', detail: 'implementer ↔ validator per pass; interim review every 3 cleared passes' },
    { title: 'Verify', detail: 'final build + test + lint gate against the staged tree' },
    { title: 'Audit', detail: 'no-deferral gate + completion auditor' },
  ],
}

// Input guard. Normalize a string payload (the args channel may stringify a
// large payload) and fail loud if `passes` is missing.
let input = args
if (typeof input === 'string') {
  try { input = JSON.parse(input) }
  catch (e) {
    throw new Error(`execute-plan-engine: args arrived as unparseable string (${e.message}); expected { passes:[…], planPath, specPath }.`)
  }
}
if (!input || typeof input !== 'object' || !Array.isArray(input.passes) || input.passes.length === 0) {
  const shape = input && typeof input === 'object'
    ? `an object with keys [${Object.keys(input).join(', ')}]`
    : `a ${typeof args} value`
  throw new Error(`execute-plan-engine: no passes to run — args resolved to ${shape}. Hand in { passes:[…], planPath, specPath }; if a large payload was dropped or stringified at the args seam, embed the passes in the script instead.`)
}

const passes = input.passes
const planPath = input.planPath
const specPath = input.specPath
const priorCleared = input.priorCleared || 0

// Interim review cadence: after every Nth cleared pass (counting passes
// cleared before a resume), a holistic reviewer ↔ fixer checkpoint sweeps
// the accumulated uncommitted work. Amortizes the final review-work cycle.
const REVIEW_CADENCE = 3

// Schemas

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

// The validator argues from spec intent that the pass does NOT deliver. It
// is disinterested — it has no stake in advance — and defaults toward the
// harder verdict for the implementer.
const VALIDATOR_SCHEMA = {
  type: 'object',
  required: ['verdict'],
  properties: {
    verdict: { enum: ['delivers', 'does_not_deliver'] },
    critique: { type: 'string' },   // populated when does_not_deliver: why the pass fails
    evidence: { type: 'string' },   // file:line citations from the diff
    falsifierHolds: { type: 'boolean' },  // does the pass's Falsifier brief actually hold against the diff?
  },
}

// One agent on BLOCKED, replacing the previous blocked-check skeptic + the
// strategist. Confirms or rejects the BLOCKED; if rejected, supplies a
// different approach.
const STRATEGIST_SCHEMA = {
  type: 'object',
  required: ['genuine'],
  properties: {
    genuine: { type: 'boolean' },
    rootCause: { type: 'string' },
    approach: { type: 'string' },
    humanAsk: { type: 'string' },
  },
}

// No-deferral audit: scans the diff for un-built work (stubs, TODOs,
// no-ops, etc). Each finding is a blocker until cleared.
const NO_DEFERRAL_SCHEMA = {
  type: 'object',
  required: ['findings'],
  properties: {
    findings: {
      type: 'array',
      items: {
        type: 'object',
        required: ['location', 'kind'],
        properties: {
          location: { type: 'string' },  // file:line
          kind: { type: 'string' },      // stub / no-op / TODO / deferred / etc.
          excerpt: { type: 'string' },
          why: { type: 'string' },       // why this is a deferral, not a legitimate WIP
        },
      },
    },
  },
}

// Interim review checkpoint: a holistic reviewer sweeps the accumulated
// uncommitted work every REVIEW_CADENCE cleared passes. Every finding goes
// to the fixer — no severity triage.
const INTERIM_REVIEW_SCHEMA = {
  type: 'object',
  required: ['findings'],
  properties: {
    findings: {
      type: 'array',
      items: {
        type: 'object',
        required: ['location', 'problem'],
        properties: {
          location: { type: 'string' },  // file:line
          problem: { type: 'string' },   // what's wrong
          why: { type: 'string' },       // why it matters
          fix: { type: 'string' },       // how to fix
        },
      },
    },
  },
}

// Completion auditor: produces the four-section report (proofs walked /
// decisions kept / decisions diverged / coverage divergences).
const COMPLETION_AUDIT_SCHEMA = {
  type: 'object',
  required: ['path'],
  properties: {
    path: { type: 'string' },
    proofsExhibited: { type: 'integer' },
    decisionsKept: { type: 'integer' },
    decisionsDiverged: { type: 'integer' },
    coverageGaps: { type: 'integer' },
    intentDrifts: { type: 'integer' },
    danglingAnnotations: { type: 'integer' },
  },
}

// Final verification gate. The verifier reads the project's own docs to
// discover the verification command set (no hardcoded commands — keeps the
// skill project-agnostic) and runs every command. A failure escalates as
// work-stuck rather than advancing to the audit phase on a broken tree.
const VERIFY_SCHEMA = {
  type: 'object',
  required: ['passed'],
  properties: {
    passed: { type: 'boolean' },
    commandsRun: { type: 'array', items: { type: 'string' } },
    failureSummary: { type: 'string' },
    failures: { type: 'array', items: { type: 'string' } },
  },
}

// Formats a pass's story slug(s) for prose: "STORY-a" or "STORY-a, STORY-b".
// A shared acceptance pass carries a comma-separated slug list (see the
// parse shape in the skill's step 2).
function storyRefs(p) {
  return String(p.storySlug || '').split(',').map(s => s.trim()).filter(Boolean)
    .map(s => `STORY-${s}`).join(', ')
}

function implementerPrompt(p, idx, total, priorFailure) {
  const hasSpec = specPath && specPath !== 'none'
  const acceptanceNote = p.isAcceptancePass
    ? `\nThis is an **acceptance pass** for ${storyRefs(p)}. Each story carries Acceptance, Falsifier, and Proof fields — read them in the spec. A story names what the user does and observes, in user terms; the delivery surface (CLI verb / HTTP route / wire message / job) is in the spec's relevant TD, not in the story. Your job is BOTH the integrating wiring AND, per story, the proof artifact (demo / example / executable proof) that story's Proof field specifies — a shared pass never merges artifacts away. Each artifact boots the real assembled product through the delivery surface the TDs prescribe, drives its story's Acceptance, exhibits the observable outcome, and uses the real value-delivering component — no stubs.`
    : ''
  const retry = priorFailure
    ? `\n## Note from the orchestrator (prior dispatch did not clear)\n${priorFailure}\nThe working tree already holds the staged work from the prior dispatch(es) on this pass. Inspect the current state first and fix forward — repair or re-run what failed, but do NOT blindly redo a task that already landed (a non-idempotent edit applied twice corrupts the tree).\n`
    : ''
  return `You are the implementer for one pass of a plan, in a fresh context. The user designed this plan and trusts you to deliver this pass; the spec is the record of intent and the plan is the route — when wording is imperfect, build what the spec wants.

## Plan
Path: ${planPath}
## Spec (source of intent)
${hasSpec ? 'Path: ' + specPath : 'No separate spec file — the plan is the record of intent.'}

## This pass: Pass ${p.number} — ${p.name}
Goal: ${p.goal}
Falsifier (the observation that would mean this pass quietly failed to deliver): ${p.falsifier}
Position: pass ${idx + 1} of ${total}.
${acceptanceNote}${retry}
## Tasks in this pass (verbatim)
${(p.tasks || []).map(t => `### Task ${t.number}\n${t.text}`).join('\n\n')}

## Discipline
- The user is taking your DONE at face value. They designed this plan, but they will not re-read your diff line by line, and a subtle gap — a handler that exists but isn't wired, a test that passes without exercising the story — is exactly what they can't catch. You are the last set of eyes on this work. Before returning DONE, re-read your own diff against the pass goal and the Falsifier above, the way you'd check work for someone who depends on it being right.
- Read files before editing — the tree may already hold work from earlier tasks or a prior dispatch of this pass.
- Deliver THIS pass only; do not do work that belongs to other passes.
- Default to rigor on any tradeoff the plan/spec leaves open (correctness, completeness, durability, atomicity, no-data-loss over the cheaper shape); leave a code comment naming the property you protected.
- **Completeness is the floor; the only divergence is overshoot.** Deliver the full intent of this pass's tasks and the spec stories they serve. When something is ambiguous or under-specified, resolve it by the necessity test: build whatever is *necessary* for the relevant user-outcome to actually hold — including pieces the spec never spelled out (the proto wiring, the emit site, the registration that does real work) — and nothing *adjacent* the stories do not require. NEVER defer, narrow, stub, no-op, or leave a "TODO / out of scope / later pass" in place of real work; a deferral is not a legal outcome — it ships a gap the user will only discover when the feature silently doesn't work for them. If you genuinely cannot finish a piece, that is a BLOCKED (below), not a silent drop.
- The pass leaves the tree in a working state — never broken.
- Checkpoint after every task with \`git add -A\` (staging, not committing). NEVER revert/reset/stash/clean the tree, even one file — fix forward by editing. Do NOT commit.

## Design-doc mutations (if any task here touches them)
If a task mutates a file under \`.ok-planner/design/concepts/\`, \`design/stories/\`, \`design/decisions/\`, or \`design/tensions/\`, keep the body self-contained AND current-state only: no file paths, no \`code:\`/\`pkg:\` citations, no external-doc references (\`docs/...\`, READMEs, sibling-repo paths), no quoted code or lint allowlists, no "Owns / Does NOT own" sections naming code paths, no \`## Notes\` / \`## History\` / \`## Changelog\` section, no dated audit-trail entries, no "previously called X" / "used to be Y" / "changed per spec Z" lines, no forward-looking "TODO" / "deferred" / "will be replaced" content. Allowed citations: other artifact slugs (concept / story / decision) and annotation IDs (\`@blessed-invariant: N\`, \`@story:\`, \`@decision:\`). Rewrite each affected section to describe the artifact as it now stands; git carries the lineage. If a task's wording leaks a path, an audit-trail line, or any backward/forward-looking framing, rewrite it as you implement — that counts as a divergence the completion auditor will record.

## BLOCKED — the only two reasons to stop short
1. Credentials/access only the user can provide. 2. An unauthorized destructive/irreversible action. Anything else ("needs discussion", "the plan was wrong", "more coupled than expected", "I'd like approval") is NOT blocked — make the call from spec intent and deliver. A review critique you disagree with is NOT a block either — address it and deliver.

## Your return value (structured)
Return the report object: status DONE when every task is done and the work fully satisfies the pass's Falsifier brief; BLOCKED with the bullet + detail only if one genuinely applies; OFF_PATTERN if you could not deliver and it is not a valid BLOCKED. List each task's number and done-state.`
}

function validatorPrompt(p) {
  const hasSpec = specPath && specPath !== 'none'
  const storyContext = p.isAcceptancePass
    ? `\n\nThis is an **acceptance pass** for ${storyRefs(p)}. Read each story in the spec at ${specPath} — it carries Acceptance (a user action producing an observable outcome, in user terms), Falsifier (the user-observable absence that proves the story not delivered), and Proof (the artifact form the implementer was told to produce). For a shared pass, judge EVERY story it serves: each must have its own proof artifact and its own observable assertion — one story exhibited does not clear the others.`
    : ''
  return `You are the **validator** for one pass of a plan. Your job is adversarial: argue that the pass does NOT deliver. Default to disbelief; require concrete evidence in the diff before agreeing.

## Plan
Path: ${planPath}
## Spec
${hasSpec ? 'Path: ' + specPath : 'No separate spec file — the plan is the record of intent.'}

## This pass: Pass ${p.number} — ${p.name}
Goal: ${p.goal}
Falsifier brief: ${p.falsifier}${storyContext}

## What you do

1. **Read the spec** (and, for acceptance passes, the story in detail). Understand the intent.

2. **Read the diff** (\`git diff\`, \`git diff --staged\`, \`git status\` for untracked). Read modified/new files where the diff lacks context.

3. **Did the implementer meet the pass's goal?** Trace the goal through the code. The work may be present but disconnected — a handler exists but isn't registered, an event emits but no subscriber consumes, a row writes but no reader observes.

4. **Did the change introduce bugs or regressions?** A diff that changes a shared surface (a route, a type field, a symbol, a contract) has callers, importers, clients, and adjacent components that depend on it. Sweep the codebase for every dependent site and verify the implementer swept them too. A dependent site that wasn't updated is a regression the pass introduced — even when nothing in the spec or the Falsifier named it.

5. **For acceptance passes: check the proof.** Has the implementer produced the named proof artifact? Does it boot the real assembled product through the real entry point, or construct components in-process and call them directly? (The latter is hollow.) Is the value-delivering component real and doing real work, or stubbed? Would a third party reading or running the artifact conclude the story is delivered? Also: **does the proof artifact carry an \`@story:<slug>\` annotation** in a top-of-file comment? A proof file without the annotation is not linked to its story for coverage purposes — flag it as a missing annotation, even if the proof otherwise exhibits the story correctly. The plan's proof task should have directed the implementer to include it; if it didn't, the validator catches it here.

5a. **For any pass that touches a \`@story:<slug>\`-annotated file**: confirm the touch is accounted for. A modification to a \`@story:<slug>\`-annotated file is acceptable when (i) the spec's \`## Proof changes\` section has an A/B/C entry for the story, OR (ii) the modification is ambient (a refactor that incidentally touches the file's call site without changing what it exhibits — same value-delivering component, same observable outcome, same Acceptance trigger). If neither holds — the diff touches a protected file and the spec didn't enumerate it and the touch is not ambient — the pass does NOT deliver. This catches off-spec proof modifications at validator time, before they ship.

5b. **For any pass that modifies files under \`.ok-planner/design/\`**: audit each modified artifact body (\`design/concepts/<slug>.md\`, \`design/stories/<slug>.md\`, \`design/decisions/<slug>.md\`, \`design/tensions/<slug>.md\`) against the canonical artifact rules. The rules canonically live in \`skills/_shared/artifact-definitions.md\` — for the validator's purposes, the mechanically-checkable subset is:

  **SELF-CONTAINMENT-RULE** (concept/story/decision bodies — grep against the diff for):
  - File or directory paths: \`\\b\\w+/\\w+\\.\\w+\\b\` (any \`foo/bar.ext\` form), \`\\bcode:\\`, \`\\bpkg:\\`, \`docs/\`, \`README\`, \`CHANGELOG\`, sibling-repo paths.
  - Citation forms: \`code:foo.go::Symbol\`, \`pkg:github.com/...\`, bare URLs (\`https?://\`).
  - Quoted code blocks (triple-backtick) inside artifact bodies (the templates have them, but a written concept body should not).
  - \`## Owns\` / \`## Does NOT own\` section headers.

  **CURRENT-STATE-ONLY-RULE** (concept/story/decision/tension bodies):
  - \`## Notes\` / \`## History\` / \`## Changelog\` section headers.
  - Dated audit-trail lines: \`^\\s*\\d{4}-\\d{2}-\\d{2}\\s+[—–-]\\s\`.
  - Backward-looking phrases: \`previously called\`, \`used to live\`, \`was tightened per spec\`, \`\\(was\\s+\\S+\\)\`, \`changed per spec\`.
  - Forward-looking phrases: \`we plan to\`, \`will be replaced\`, \`TODO:\`, \`deferred\`, \`open question for later\`.

  **TENSION-SURFACE-RULE** (tension bodies only):
  - Inside \`## Resolution candidates\` section: any of the SELF-CONTAINMENT-RULE patterns above are violations. \`## What is muddy\` and \`## Evidence\` may cite paths.

  **PROOF-PROTECTION-RULE** (when the diff includes story-file mutations):
  - A story-file \`Proof:\` field change must trigger an audit of every \`@story:<slug>\`-annotated file in the codebase — confirm each still satisfies the new Proof field text.

  Any violation found is a pass-blocking issue — the implementer wrote design-doc content that violates the canonical rules. The fix is to rewrite the offending lines as path-free, current-state-only prose. The validator surfaces each violation as a critique line (file:line + which rule + the offending text); the implementer fixes on next dispatch.

  This check runs at validator time precisely so drift does not accumulate until cycle 2 / \`/review-design\` has to clean it up — by then there may be dozens of edits in the diff and the originating context is gone. Catching at pass close means same-turn fix with full implementer context.

6. **Check the Falsifier brief.** Does the absence the Falsifier names hold against the diff? If you can point at the code and show the absence is real, the pass does NOT deliver.

## Adversarial defaults
- Default to **does_not_deliver**. Confirm with **delivers** only when you've actively tried to find evidence the pass falls short and cannot.
- Cite specific file:line evidence in the diff for whichever verdict you reach. "Looks reasonable" is not a verdict.
- The Falsifier names the spec author's minimum, not your ceiling. A dependent site outside the Falsifier's named scope is still a finding.
- Hollow proofs are the canonical failure mode for acceptance passes: shape checks, registration assertions, in-process construction calling handlers directly, canned-success stubs at the value-delivering component. Catch them.

## Your return value
Return the verdict object:
- verdict: "delivers" or "does_not_deliver"
- critique: when does_not_deliver, a sharp specific critique the implementer can act on (no platitudes)
- evidence: file:line citations supporting your verdict
- falsifierHolds: true if you can show the Falsifier's observable absence is real in the diff; false if you cannot
`
}

function strategistPrompt(p, report) {
  const hasSpec = specPath && specPath !== 'none'
  return `An implementer reported BLOCKED on Pass ${p.number} — ${p.name}. You are the strategist: decide, skeptically, whether the BLOCKED is genuine, and — if not — supply a different approach the next implementer should try. Default to NOT genuine.

## The two valid BLOCKED conditions
1. Credentials or access only the user can provide (a secret, an API key, a remote resource) with no workaround.
2. An unauthorized destructive or irreversible action the plan does not authorize.

Anything else — "needs discussion," "the plan was wrong about X," "more coupled than expected," "I'd like approval," "a validator critique I disagree with" — does NOT qualify. Default to genuine: false.

## The implementer's report
${report.status === 'BLOCKED' ? `Bullet cited: ${report.blockedBullet || '(none given)'}\nDetail: "${report.blockedDetail || ''}"` : `Status was ${report.status} but you're being run anyway — treat this as a non-genuine block and propose a different approach.`}

## Plan
Path: ${planPath}
## Spec
${hasSpec ? 'Path: ' + specPath : 'No separate spec file.'}

## This pass
Goal: ${p.goal}
Falsifier: ${p.falsifier}

## What you do
Read the pass, the spec, and the CURRENT working tree (the prior implementer's staged work is there).

- If the situation plainly fits bullet 1 or 2 above, set genuine: true and state exactly what the user needs to provide as humanAsk.
- Otherwise, set genuine: false. Diagnose the **root cause** the prior implementer hit — the underlying reason it stalled, not the surface symptom. Then prescribe a concrete, DIFFERENT approach: specifically what to do differently this time, not "try harder" and not a restatement of the plan.

## Your return value
- genuine: bool (default false unless bullet 1 or 2 plainly applies)
- rootCause: one-line read of why the prior attempt stalled (when not genuine)
- approach: the different approach to try (when not genuine)
- humanAsk: the specific user input needed (when genuine)
`
}

function interimReviewPrompt(passesCleared, passesTotal) {
  const hasSpec = specPath && specPath !== 'none'
  return `You are the **interim reviewer** for a plan mid-execution. ${passesCleared} of ${passesTotal} passes have cleared; the rest have NOT run yet. Your job is a holistic code review of the work delivered so far, so quality issues get fixed now instead of piling up to the end of the run.

## Context
Plan: ${planPath}
${hasSpec ? 'Spec: ' + specPath : 'No separate spec file — the plan is the record of intent.'}

## Scope — the accumulated uncommitted work
All of the plan's work so far is uncommitted. Run \`git status\`, \`git diff\`, \`git diff --staged\` (untracked files included) before forming any findings. Read modified and new source files in their entirety for context. Working-tree state is your subject.

## Review focus
- Correctness: bugs, edge cases, off-by-one errors
- Safety: data loss, security issues, resource leaks, irreversible operations
- State integrity: can we get stuck in a state? Double-execute? Skip steps?
- Load-bearing properties: name the properties the spec/plan depend on (durability, atomicity, ordering, idempotency, no-data-loss) and verify the code actually guarantees each — not just on the happy path
- Cross-pass drift: the per-pass validators each saw one pass; you see all of them together. Duplicated helpers, inconsistent naming or contracts between passes, a later pass quietly diverging from a convention an earlier pass set
- Dead code, unused imports, stale comments
- Pre-existing issues in any file you read — report them too

## What NOT to flag — this is a mid-run review
- Work that belongs to passes that have not run yet. The plan is ${passesCleared}/${passesTotal} done; absence of future-pass work is the expected state, not a finding. Only flag a gap if it falls inside the passes already cleared.
- Spec-completeness sweeps (stubs, TODOs, hollow proofs) beyond what you naturally hit — the no-deferral audit runs that systematically at the end.
- Design-doc compliance under \`.ok-planner/design/\` — the final review handles it.

## Output
Do NOT categorize by severity — every issue you list goes to a fixer. Return the findings array; for each: location (file:line), problem, why it matters, how to fix. Return an empty findings array only when the accumulated work is genuinely clean.
`
}

function interimFixPrompt(findings) {
  const list = findings.map((f, i) => `${i + 1}. ${f.location} — ${f.problem}\n   Why: ${f.why || '(unstated)'}\n   Suggested fix: ${f.fix || '(reviewer left the fix to you)'}`).join('\n')
  return `The interim reviewer found ${findings.length} issue(s) in the plan's accumulated work. Fix every one of them — no triage, no deferral.

${specPath && specPath !== 'none' ? `Use the spec at ${specPath} as the source of intent.` : `Use the plan at ${planPath} as the source of intent.`}

## Findings to fix
${list}

## Discipline
- Read each cited file before editing.
- Fix the underlying problem, not the symptom the reviewer quoted.
- Stay inside the work already delivered — do NOT build work that belongs to passes that have not run yet, even if a fix tempts you toward it.
- Do NOT introduce stubs, TODOs, or deferrals while fixing.
- Leave the tree in a working state.
- Checkpoint with \`git add -A\` after each fix. NEVER revert/reset/stash/clean; do NOT commit.

When every finding is fixed, report DONE.
`
}

function noDeferralAuditPrompt() {
  const specRead = specPath && specPath !== 'none' ? `spec ${specPath}; ` : ''
  return `You are the **no-deferral auditor**. The implementer just finished the plan. Your job is to find any place where promised work was deferred, stubbed, or otherwise not actually delivered — and surface it as a blocker, not a divergence.

Read: plan ${planPath}; ${specRead}the diff (\`git diff\`, \`git diff --staged\`, \`git status\` for untracked); and modified/new files where the diff lacks context.

## What counts as a deferral (a blocker)
Each of these is a finding the run does NOT advance past until cleared:

- A \`TODO\` / \`FIXME\` / \`XXX\` marker on a code path the spec depends on.
- A "later," "for now," "out of scope," "deferred," "not implemented," "stub," or "placeholder" comment in a function/handler/method the spec depends on.
- A handler / endpoint / class registered or declared but doing nothing (\`return nil\`, \`pass\`, \`{ /* TODO */ }\`).
- An error class declared but never emitted, an event type declared but never published.
- A config flag or field accepted but ignored.
- A function whose body is a hard-coded canned return when the spec asks for it to do real work.
- An acceptance pass's proof artifact missing, empty, or trivially passing (asserts \`true\`, returns \`200\` without exercising the story).
- Any \`raise NotImplementedError\` / \`panic("unimplemented")\` / \`throw new Error("not yet")\` on a spec-dependent path.

## What does NOT count (do not flag)
- Tests / examples that the spec did not require (these are normal absences, not deferrals).
- Test helpers or fixtures that intentionally short-circuit (a fixture's job is to be canned).
- Comments noting historical decisions or alternatives considered.
- Code paths that are intentionally NOT part of the spec.

## Your return value
List every finding. For each:
- location: file:line
- kind: stub / no-op / TODO / placeholder / canned / hollow-proof / etc.
- excerpt: the literal text from the code
- why: why this is a deferral and what the spec requires there

Return an empty findings array only when the diff is genuinely deferral-free.
`
}

function noDeferralFixerPrompt(findings) {
  const list = findings.map((f, i) => `${i + 1}. ${f.location} — ${f.kind}: ${f.why || '(no rationale)'}\n   Excerpt: ${f.excerpt || ''}`).join('\n')
  return `The no-deferral auditor found ${findings.length} deferral(s) in the just-completed plan. Each is a completion failure that must be cleared before the run advances. Fix every one of them.

## Findings to clear
${list}

## What to do
For each finding, finish the work the deferral was avoiding. ${specPath && specPath !== 'none' ? `Use the spec at ${specPath} as the source of intent.` : 'Use the plan as the source of intent (no separate spec file).'} Build whatever is *necessary* for the spec's stories to actually hold — including pieces the spec did not literally name (the necessity rule).

## Discipline
- Read each cited file before editing.
- Do NOT change the structure of the deferral marker itself ("change the TODO to DONE"); fix the underlying work the marker stood in for.
- Default to rigor on any tradeoff (correctness, completeness, durability).
- Checkpoint with \`git add -A\` after each fix.
- Do NOT introduce new deferrals while fixing these.

When you have cleared all findings, report DONE. The audit will re-run; any remaining deferral re-dispatches you with the residual list.
`
}

function verifierPrompt() {
  const specRead = specPath && specPath !== 'none' ? `spec ${specPath}; ` : ''
  return `You are the **final verifier**. Every pass has cleared its implementer ↔ validator loop. Your job is to run the project's full verification suite against the staged tree and confirm nothing is broken.

## What you do

1. **Identify the project's verification command set.** The skill is project-agnostic — it does not hardcode commands. Discover them by reading, in order, until you have a complete set:
   - The plan at ${planPath} — its plan-wide notes / "Verification commands" / "After each pass" section.
   - ${specRead}any verification section it carries.
   - The project's CLAUDE.md (look for "After Code Changes", "Verification", or analogous sections).
   - The project's Makefile / package.json / scripts directory for canonical aggregate targets (e.g. \`make verify\`, \`make test-all\`, \`npm test\`, \`go test ./...\`, \`pytest\`, \`cargo test\`).

   A typical set includes: build / type-check; unit tests; lint; race / concurrency-sensitive tests where the project's docs name them; integration / scenario / e2e tests if the plan touched code under their scope.

2. **Run every command in the set.** Use Bash. Capture output. Long-running commands are fine — set generous timeouts; this is the final gate.

3. **Read each command's output.** Distinguish *real failures* (build errors, test failures, lint errors, race detections) from *expected output* (test summaries, "X tests passed", warnings the project's docs say to ignore, intentionally skipped tests).

4. **If any command fails:** gather the failure context (failing file:line, error message, the relevant diff context that introduced it where you can identify it). Return passed=false with a one-line failureSummary plus a per-failure list.

5. **If everything passes:** return passed=true with the list of commands you ran.

## What NOT to do
- Do NOT fix failures. Your job is to detect, not repair. A failure escalates the workflow as work-stuck so a human can decide.
- Do NOT skip a command because you think it's "already covered" by another.
- Do NOT substitute a faster command for one named in the project's docs.
- Do NOT commit or modify the tree.

## Your return value
- passed: boolean — true iff every command in the set returned success
- commandsRun: array of strings — the commands you actually ran, in order
- failureSummary: one-line summary of what failed (omit when passed=true)
- failures: array of strings, one per failing check, with file:line where available (omit when passed=true)
`
}

function completionAuditPrompt() {
  const specRead = specPath && specPath !== 'none' ? `spec ${specPath}; ` : ''
  return `You are the **completion auditor**. The plan is complete, the verification gate passed, and the no-deferral gate is clean. Your job is to produce a four-section report covering 100% of the spec PLUS coverage divergences surfaced by the closing audit: every story's proof exhibited working, every technical decision in the spec accounted for as kept or diverged, every proof-coverage divergence surfaced.

Read: plan ${planPath}; ${specRead}the spec's \`## Manifest\` section (the contract you walk); the diff; and any proof artifacts the implementer produced (demo scripts, example files, executable proofs).

Before writing the report, run the **closing coverage + intent-drift audit**:

**Coverage check (whole-corpus, cheap).** For every live story in \`.ok-planner/design/stories/<slug>.md\`, grep the codebase for \`@story:<slug>\` annotations (\`rg -n '@story:\\s*<slug>'\`). Record every story with **zero** matches — that story has no proof artifact (or its proof was removed, or its annotation drifted).

**Intent-drift check (spec-scoped, judgment).** For every story the spec touched (mutated under \`## Design changes\` or named in \`## Proof changes\`) AND every \`@story:<slug>\`-annotated file modified in the diff: read the proof file and read the story's current \`Proof:\` field. Judge whether the proof still satisfies the story's Proof field. Verdict per artifact: **satisfies** / **does not satisfy** / **uncertain**. Cite file:line for any "does not satisfy" or "uncertain" verdict — name what the Proof field requires that the artifact does not exhibit.

**Annotation integrity (cheap).** \`rg -n '@story:\\s*\\S+'\` across the codebase. For every annotation, confirm the slug resolves to a live \`stories/<slug>.md\` file. Annotations pointing at retired or missing story slugs are dangling — record them.

Findings from these three checks land in section 4 of the report.

Write the report to ${planPath.replace(/\.md$/, '')}-completion-report.md with these four sections in order:

## 1. Proof walkthrough
For every story in the spec's manifest:
- Story slug + one-line restatement.
- The proof artifact's path (the file the implementer produced).
- A summary of what the artifact exhibits (what running or reading it shows).
- The artifact's invocation (the command to run it, or the path to read it).
- Whether the artifact carries the \`@story:<slug>\` annotation (it must — flag absence as a process defect).
- Status: EXHIBITS WORKING (the artifact runs / reads as proving the story) or GAP (the artifact is missing, hollow, or exhibits something other than the story). A GAP here is unexpected — the no-deferral audit should have caught it; flag it as a process defect if present.

## 2. Technical decisions kept
For every TD in the spec's manifest **that the implementation honored as the spec specified**:
- TD slug + one-line restatement of the Choice.
- File:line citation showing the decision is embodied in the code (the persistence call, the registration mechanism, the chosen cadence).

**Explicit enumeration — every spec TD lands either here or in section 3.** No silent attestation. The audit's value is forcing a full walk.

## 3. Technical decisions diverged
For every TD in the spec where the implementation took a different shape than the spec said, AND every implementation choice the spec did not anticipate (necessitated by the necessity rule). For each:
- TD slug + one-line restatement.
- What the spec said vs. what was implemented (file:line where useful).
- Flavor: **improved** (implementer found a better shape), **selected** (spec left a choice open and the implementer picked), or **necessitated** (work the spec did not name but the necessity rule required for a story to hold).
- Reason: one or two sentences.

## 4. Coverage divergences
For every finding from the closing coverage + intent-drift audit. Each entry is one of three kinds:

- **Coverage gap** — a live story with zero \`@story:<slug>\` annotations in the codebase. Name the story slug; note whether the spec touched the story (if so, the validator should have caught this — process defect); name what's needed to resolve (restore a proof artifact, or open a brainstorm to deprecate the story).
- **Intent drift** — a \`@story:<slug>\`-annotated proof file whose content no longer satisfies the story's \`Proof:\` field. Quote the relevant excerpt from the story's Proof field; quote what the proof actually exhibits; name file:line. Note whether the spec had a \`## Proof changes\` entry for this story (if so and the entry was "A. Preserve" but the proof drifted, that's a process defect — the validator should have caught it).
- **Dangling annotation** — an \`@story:<slug>\` annotation pointing at a slug that doesn't resolve to a live story file. Name the annotation site and the unresolved slug. Note whether the spec removed the story (if so, the implementer should have removed all annotations too — process defect).

Each entry includes a brief recommendation: **accept as informational** (the divergence is known and intentional, just record it), **course-correct now** (open a brainstorm to address — e.g., restore the proof, deprecate the story, rewrite the annotation), or **bounce back to the implementer** (a process defect the implementer should fix before close).

A section with no findings should still be present in the report, with the text "No coverage divergences." — explicit zero is a stronger signal than absence.

## Coverage check (at the bottom of the report)
A short summary: stories exhibited / total in manifest; TDs kept + TDs diverged should equal total TDs in manifest; coverage divergences by kind (gaps / drifts / dangling). Flag any mismatch as a process defect.

## Your return value
Return { path: <the report path you wrote>, proofsExhibited, decisionsKept, decisionsDiverged, coverageGaps, intentDrifts, danglingAnnotations }.
`
}

// ----------------------------------------------------------------------------
// Main loop
// ----------------------------------------------------------------------------

phase('Execute')

// Defensive validation: every pass must have a falsifier (the validator's
// contract). The skill is supposed to catch this in step 2; the engine
// re-checks to fail loud rather than dispatch a validator with no contract.
for (let i = 0; i < passes.length; i++) {
  const p = passes[i]
  if (!p.falsifier || String(p.falsifier).trim() === '') {
    return { status: 'escalate', kind: 'invalid-plan', pass: p.number,
             detail: `Pass ${p.number} has no Falsifier brief — the validator has nothing to argue against. Take this back to write-plan.` }
  }
  if (p.isAcceptancePass && (!p.storySlug || String(p.storySlug).trim() === '')) {
    return { status: 'escalate', kind: 'invalid-plan', pass: p.number,
             detail: `Pass ${p.number} is an acceptance pass but has no story slug. Take this back to write-plan.` }
  }
}

const cleared = []
for (let i = 0; i < passes.length; i++) {
  const p = passes[i]
  let priorFailure = null
  let strategistRan = false
  let passCleared = false

  for (let attempt = 1; attempt <= 2; attempt++) {
    const report = await agent(implementerPrompt(p, i, passes.length, priorFailure), {
      label: `pass-${p.number}:impl#${attempt}`, phase: 'Execute', schema: REPORT_SCHEMA,
    })

    if (!report) {
      priorFailure = 'Prior dispatch was skipped before returning a report; re-dispatching.'
      log(`pass ${p.number}: dispatch skipped on attempt ${attempt}, re-dispatching`)
      continue
    }

    if (report.status === 'BLOCKED') {
      // First BLOCKED in this pass → strategist. Second BLOCKED after a
      // strategist intervention → escalate as work-stuck (the strategist's
      // different approach didn't unstick the implementer).
      if (strategistRan) {
        return { status: 'escalate', kind: 'work-stuck', pass: p.number,
                 lastFailure: priorFailure || 'Repeated BLOCKED after the strategist intervened with a different approach. The implementer is genuinely stuck.',
                 cleared }
      }
      const strat = await agent(strategistPrompt(p, report), {
        label: `pass-${p.number}:strategist`, phase: 'Execute', schema: STRATEGIST_SCHEMA,
      })
      strategistRan = true
      if (!strat || strat.genuine) {
        return { status: 'escalate', kind: 'blocked', pass: p.number,
                 bullet: report.blockedBullet, detail: report.blockedDetail || '',
                 humanAsk: (strat && strat.humanAsk) || '',
                 cleared }
      }
      priorFailure = `You reported BLOCKED, but the strategist judged it not genuine. Root cause the prior attempt missed: ${strat.rootCause || '(unstated)'}\n\nTake this DIFFERENT approach: ${strat.approach || '(unstated)'}\n\nDeliver the pass.`
      log(`pass ${p.number}: BLOCKED → strategist intervened with different approach`)
      // Strategist intervention is a routing fix, not an implementer attempt — give the
      // new framing one full dispatch by canceling this iteration's counter advance.
      // (Without this, a BLOCKED arriving on the final attempt would exit the loop
      // before the strategist's approach could be tried.)
      attempt--
      continue
    }

    if (report.status !== 'DONE') {
      priorFailure = `Prior dispatch returned ${report.status} — not a clean DONE and not a valid BLOCKED. Deliver the pass.`
      log(`pass ${p.number}: off-pattern on attempt ${attempt}, re-dispatching`)
      continue
    }

    // DONE claimed → confirm every task in the pass is marked done.
    const doneNumbers = new Set((report.tasks || []).filter(t => t.done).map(t => Number(t.number)))
    const missing = (p.tasks || []).filter(t => !doneNumbers.has(Number(t.number))).map(t => t.number)
    if (missing.length) {
      priorFailure = `Report claimed DONE, but task(s) ${missing.join(', ')} are not marked done in the per-task list. Every task in the pass must be delivered.`
      log(`pass ${p.number}: DONE claimed with undone task(s) ${missing.join(', ')}, re-dispatching`)
      continue
    }

    // Validator — the adversarial judge.
    const verdict = await agent(validatorPrompt(p), {
      label: `pass-${p.number}:validator#${attempt}`, phase: 'Execute', schema: VALIDATOR_SCHEMA,
    })

    if (verdict && verdict.verdict === 'delivers') {
      passCleared = true
      log(`pass ${p.number}: validator confirmed delivery`)
      break
    }

    priorFailure = `Your DONE was reviewed and rejected. The pass does NOT deliver because:\n\n${(verdict && verdict.critique) || '(no critique returned)'}\n\nEvidence cited:\n${(verdict && verdict.evidence) || '(none)'}\n\nThe pass's Falsifier brief is: ${p.falsifier}\n\nAddress the critique and re-deliver.`
    log(`pass ${p.number}: validator rejected on attempt ${attempt}, re-dispatching`)
  }

  if (!passCleared) {
    return { status: 'escalate', kind: 'work-stuck', pass: p.number,
             lastFailure: priorFailure,
             note: 'Two implementer attempts (with validator critiques between) could not deliver this pass. New input from you is needed; lastFailure carries the validator\'s most recent critique or the strategist\'s different approach that was tried and still failed.',
             cleared }
  }
  cleared.push(p.number)
  log(`pass ${p.number} cleared (${cleared.length}/${passes.length})`)

  // Interim review checkpoint. Every REVIEW_CADENCE cleared passes (counted
  // across resumes via priorCleared), sweep the accumulated work with a
  // reviewer ↔ fixer loop. Best-effort, NOT a gate: residual findings flow
  // to the final skill-side review-work anyway, so non-convergence logs and
  // continues rather than escalating. Skipped after the final pass — the
  // final review runs immediately afterward, so an interim sweep there is
  // redundant (a pass count not divisible by the cadence is fine for the
  // same reason).
  const totalCleared = priorCleared + cleared.length
  if (totalCleared % REVIEW_CADENCE === 0 && i < passes.length - 1) {
    let residual = -1
    for (let cycle = 1; cycle <= 2; cycle++) {
      const review = await agent(interimReviewPrompt(totalCleared, priorCleared + passes.length), {
        label: `interim-review@${totalCleared}#${cycle}`, phase: 'Execute', schema: INTERIM_REVIEW_SCHEMA,
      })
      if (!review) {
        log(`interim review after ${totalCleared} passes: reviewer returned no result; continuing`)
        break
      }
      if (!review.findings || review.findings.length === 0) {
        residual = 0
        break
      }
      residual = review.findings.length
      log(`interim review after ${totalCleared} passes: ${review.findings.length} issue(s); dispatching fixer`)
      await agent(interimFixPrompt(review.findings), {
        label: `interim-fix@${totalCleared}#${cycle}`, phase: 'Execute',
      })
    }
    if (residual === 0) log(`interim review after ${totalCleared} passes: clean`)
    else if (residual > 0) log(`interim review after ${totalCleared} passes: fix cycles exhausted with issue(s) outstanding; the final review will converge them`)
  }
}

phase('Verify')

// Final verification gate. The verifier dispatches once and reports
// passed=true or passed=false. Failure escalates as work-stuck so the user
// can decide (fix and resume, accept and override, abandon). The skill is
// project-agnostic so we do NOT run hardcoded commands here — the verifier
// reads the project's docs to discover its own command set.
const verifyResult = await agent(verifierPrompt(), {
  label: 'verify', phase: 'Verify', schema: VERIFY_SCHEMA,
})

if (!verifyResult || !verifyResult.passed) {
  return { status: 'escalate', kind: 'work-stuck', pass: -1,
           lastFailure: `Final verification gate failed: ${(verifyResult && verifyResult.failureSummary) || '(verifier did not return a result)'}.\n\nCommands run: ${(verifyResult && verifyResult.commandsRun || []).join(', ') || '(unknown)'}\n\nFailures:\n${(verifyResult && verifyResult.failures || []).map(f => `- ${f}`).join('\n') || '(no per-failure detail returned)'}`,
           cleared }
}

log(`verification gate passed: ${(verifyResult.commandsRun || []).length} command(s)`)

phase('Audit')

// No-deferral audit as gate: loops auditor → fixer → re-audit until clean.
// Capped at 3 cycles; if the auditor is still finding deferrals at cycle 3,
// escalate as work-stuck on the residual list.
let lastFindings = null
for (let cycle = 1; cycle <= 3; cycle++) {
  const audit = await agent(noDeferralAuditPrompt(), {
    label: `no-deferral-audit#${cycle}`, phase: 'Audit', schema: NO_DEFERRAL_SCHEMA,
  })
  if (!audit) {
    lastFindings = [{ location: 'unknown', kind: 'audit-error', why: 'auditor returned no result' }]
    continue
  }
  if (!audit.findings || audit.findings.length === 0) {
    lastFindings = null
    break
  }
  log(`no-deferral audit cycle ${cycle}: ${audit.findings.length} deferral(s) found; dispatching fixer`)
  await agent(noDeferralFixerPrompt(audit.findings), {
    label: `no-deferral-fixer#${cycle}`, phase: 'Audit',
  })
  lastFindings = audit.findings
}

if (lastFindings) {
  return { status: 'escalate', kind: 'work-stuck', pass: -1,
           lastFailure: `The no-deferral audit could not be cleared after 3 cycles. Residual findings:\n${lastFindings.map(f => `- ${f.location} (${f.kind}): ${f.why || ''}`).join('\n')}\n\nThe fixer was unable to close the gaps autonomously; user direction needed.`,
           cleared }
}

// Completion auditor produces the three-section report.
const completion = await agent(completionAuditPrompt(), {
  label: 'completion-audit', phase: 'Audit', schema: COMPLETION_AUDIT_SCHEMA,
})

return { status: 'complete',
         passesCleared: priorCleared + cleared.length,
         clearedThisRun: cleared.length,
         completionReport: completion || { path: 'none' } }
```

## Rules

- One implementer at a time per pass (passes depend on prior passes).
  The workflow never runs passes in parallel.
- The validator is **disinterested and adversarial** — defaults to
  "does not deliver," confirms only with concrete citation against
  the diff. Hollow proofs (shape checks, registration assertions,
  canned stubs at the value-delivering component) are the canonical
  failure mode the validator catches.
- **One strategist intervention per pass** on BLOCKED. The
  strategist either confirms the BLOCKED (escalate) or supplies a
  root-cause read and a different approach (re-dispatch). A second
  BLOCKED after a strategist intervention escalates as
  `work-stuck`.
- **Interim review checkpoint every 3 cleared passes** (counted
  across resumes). A holistic reviewer sweeps the accumulated
  uncommitted work — review-work's correctness / safety /
  state-integrity dimensions plus cross-pass drift the per-pass
  validators can't see — and a fixer clears every finding, no
  severity triage. It is an **amortizer, not a gate**: capped at 2
  reviewer ↔ fixer cycles, and residual findings log and flow to
  the final skill-side `review-work` instead of escalating. It
  never flags work belonging to passes that haven't run, and it is
  skipped after the final pass (the final review follows
  immediately; a pass count not divisible by 3 needs no special
  handling for the same reason).
- **No bare repeats.** A re-dispatch always carries new input — the
  validator's critique on attempt 2, the strategist's different
  approach on a BLOCKED retry. Raising the cap without adding
  perspective is not a fix.
- **Final verification gate runs every project-prescribed check.**
  The verifier discovers the command set from the project's own
  docs (plan / spec / CLAUDE.md / Makefile) rather than the skill
  hardcoding commands. Build, lint, unit tests, race tests on
  named race-sensitive packages, integration / scenario tests
  where the plan's scope touches them. A failure escalates as
  `work-stuck`; the verifier does not repair, the human decides.
  The implementer's per-pass tests catch local breakage; this
  gate catches cross-pass interactions and surfaces a foundational
  regression that escaped a Falsifier's named scope.
- **No-deferral audit is a gate, not a report.** It loops with a
  fixer dispatch until clean. The closing walk never surfaces a
  "what was deferred and fixed" section — by the time the user
  sees anything, the contract has been met.
- **Completion auditor produces the closing-walk artifact** — the
  three-section report (proofs working, decisions kept, decisions
  diverged) that the skill walks with the user and archives.
- **Every pass leaves the tree working.** No `broken-intentional`
  end-state, no restorer model. A plan that implies multi-pass
  teardown-then-rebuild must express it as one pass with both.
- **Never commit; never revert/reset/stash/clean** (binds the skill
  and every agent). Implementers checkpoint with `git add -A`
  (non-destructive); recovery is fix-forward.
- Escalations are the only return-to-human boundaries: `blocked`
  (strategist-confirmed credentials/destruction), `work-stuck`
  (implementer ↔ validator could not converge, the verification
  gate failed, or the no-deferral gate could not be cleared
  autonomously), `invalid-plan` (structural — missing falsifier
  or missing story slug on an acceptance pass).
