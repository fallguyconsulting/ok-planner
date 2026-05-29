---
name: ok-version
description: "ONLY activated by explicit /ok-version slash command. Never auto-triggered by conversation content."
---

# ok-planner version check

Read-only. Reports two things:

1. **Which ok-planner version is installed on disk** — the answer to "is the installed plugin the version I expect?"
2. **Whether the code of conduct governing *this live session* matches the conduct that is installed on disk** — the answer to "did I reload the plugin but keep running the old conduct?"

## Why this is a skill and not just the session-start hook

The `ok-conduct` output style is read into the system prompt **once**, at session start or `/clear`, and is **not** refreshed by `/reload-plugins`. `/reload-plugins` swaps in new skills, hooks, agents, and MCP/LSP servers — but the active output style stays whatever loaded at the last session start. So a session can be running **current skills under a stale conduct**.

The session-start hook can only report the on-disk state as of session start. This skill reads the conduct **fresh from disk at invocation** and compares it against what the session is **actually operating under**, which is the only way to catch that drift mid-session.

## Procedure

### 1. Read the installed versions from disk

Run this. It prefers `$CLAUDE_PLUGIN_ROOT` (the plugin's installed location); if that is unset it falls back to the current project (the local-path / dogfooding case) and then to searching the Claude config dir:

```bash
root="${CLAUDE_PLUGIN_ROOT:-}"
if [ -z "$root" ] || [ ! -f "$root/.claude-plugin/plugin.json" ]; then
  for cand in "$PWD" \
    "$(find "$HOME/.claude" -type d -name ok-planner -path '*plugins*' 2>/dev/null | head -1)"; do
    if [ -n "$cand" ] && [ -f "$cand/.claude-plugin/plugin.json" ]; then root="$cand"; break; fi
  done
fi

if [ -z "$root" ] || [ ! -f "$root/.claude-plugin/plugin.json" ]; then
  echo "ok-planner install not found — could not resolve the plugin root."
else
  plugin_version=$(sed -n 's/.*"version"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$root/.claude-plugin/plugin.json" | head -1)
  conduct_version=$(sed -n 's/^Conduct version:[[:space:]]*\(.*\)/\1/p' "$root/output-styles/ok-conduct.md" 2>/dev/null | head -1)
  echo "plugin_root:       $root"
  echo "installed_plugin:  ${plugin_version:-unknown}"
  echo "installed_conduct: ${conduct_version:-unknown}"
fi
```

### 2. Read the conduct version governing THIS session

Look at your own active **Output Style** instructions — the code of conduct currently in your system prompt. Find the line that begins `Conduct version:` and read it verbatim.

- If there is **no** such line, the active conduct predates version stamping (or no `ok-conduct` style is active in this session). Treat the active conduct version as `unstamped`.
- Do **not** take this number from disk, and do **not** take it from the session-start context line — both reflect the on-disk file, not what this session loaded. The whole point is to compare what you are *actually operating under* against what is on disk right now.

### 3. Report and give the verdict

Print exactly these lines, filling in the values:

- **Installed plugin version:** `<installed_plugin>`
- **Installed conduct version:** `<installed_conduct>`
- **Active conduct version (this session):** `<what you read from your own instructions, or "unstamped">`
- **Verdict:**
  - If the active conduct version equals the installed conduct version → **Conduct is current.**
  - Otherwise (including `unstamped`) → **Conduct is STALE.** This session is running `<active>`, but disk has `<installed_conduct>`. To pick up the installed conduct, run `/reload-plugins` (refreshes skills and hooks) **and then** `/clear` (re-reads the output style), or start a fresh session. `/reload-plugins` alone will not refresh the conduct.

This skill never edits files and never chains to another skill. Report and stop.
