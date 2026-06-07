---
name: verify
description: "ONLY activated by explicit /verify slash command or ok-planner pipeline. Never auto-triggered."
---

# Verification Before Completion

**Rule:** No completion claims without fresh verification evidence.

## The Gate

Before claiming any status:
1. **Identify** what command proves the claim
2. **Run** the command (fresh, complete)
3. **Read** full output, check exit code
4. **Verify** output confirms the claim
5. **Then** make the claim with evidence

## Common Requirements

| Claim | Requires | Not Sufficient |
|-------|----------|----------------|
| Tests pass | Test output: 0 failures | Previous run, "should pass" |
| Build succeeds | Build output: exit 0 | Partial check |
| Bug fixed / behavior works | The test was **red without the fix and green with it** — you ran both and saw the flip | A test that passed the first time it ran; it was never coupled to the change |
| Spec / plan complete | **Every** user-outcome story's acceptance gate is green, each run against the real assembled product | Most stories green; "the headline scenario works"; the mechanism exists but the user-outcome was never observed |

## A green test only counts if it was red first

For any change to runtime behavior, "the test passes" is not proof on its own. A test that was green before the change existed proves nothing about the change — it was never coupled to it. Proof is the **flip**: the test fails without the fix and passes with it, and you saw both.

- New behavior / bug fix: write the test first, run it, **watch it fail**, then implement until it passes. If the test passed on its very first run, it isn't testing the new behavior — fix the test, not the claim.
- Inherited a green test you never saw fail? Revert the fix (edit it out, not `git stash`/`checkout`), run the test, confirm it goes red, restore the fix. A test that stays green with the fix deleted is theater.
- The test must drive the real system and assert an observable outcome — a persisted row's state, a call made over the wire, a status returned, a terminal reached — not a struct/proto shape or a pure-helper return.

## Red Flags

If you're thinking "should work", "probably", "seems to" -- stop and run the command.

A new test that passed on its first run, for behavior that didn't work before, is not evidence — it was never red, so it never tested the fix. Make it fail without the fix before you trust it.
