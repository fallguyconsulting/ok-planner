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
| Bug fixed | Original symptom passes | Code changed, assumed fixed |

## Red Flags

If you're thinking "should work", "probably", "seems to" -- stop and run the command.
