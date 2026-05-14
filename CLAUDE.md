# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a **single-plugin Claude Code marketplace** that ships the `ok-planner` plugin: planning, implementation, and review skills (a pipeline: brainstorm → write-plan → execute-plan → review-work → verify), plus the `ok-conduct` output style.

The deliverable is markdown `SKILL.md` files, plugin and marketplace manifests, an output style, and a bash `SessionStart` hook. There is no build, no test runner, no `package.json`. Edits are almost always to skill prompts under `skills/<skill>/SKILL.md`.

## Layout

```
.claude-plugin/marketplace.json   # Marketplace manifest — single-plugin, sources the plugin at "."
.claude-plugin/plugin.json        # Plugin manifest (name/description/version)
hooks/hooks.json                  # Declares SessionStart
hooks/session-start               # Bash script; must stay executable
skills/<skill>/SKILL.md           # The skill prompt; frontmatter name/description required
output-styles/<name>.md           # Claude Code output styles shipped by the plugin (e.g., ok-conduct)
```

The plugin lives at the repo root rather than under `plugins/<name>/`; `marketplace.json` points at `"."`.

The `SessionStart` hook in `hooks/session-start` reads `skills/ok-planner/SKILL.md` at session start and injects its contents as `additionalContext` via stdout JSON. It uses a hand-rolled `escape_for_json` function — any changes to that script must preserve valid JSON output (test by running the script directly; it should emit a parseable JSON object).

## How skills are wired

Every `SKILL.md` starts with YAML frontmatter:

```
---
name: <skill-name>
description: "ONLY activated by explicit /<command> slash command. Never auto-triggered by conversation content."
---
```

The "ONLY activated by explicit slash command" phrasing is load-bearing — it prevents Claude from invoking skills inferentially. Preserve it on new skills.

Skills chain by instructing the orchestrator to invoke the next one (e.g., `brainstorm` ends by invoking `ok-planner:write-plan`). They do not chain automatically — the chain is part of the skill prompt.

## ok-planner's review discipline (important when editing skills here)

The ok-planner pipeline enforces a **no-triage** policy on code review results. Review skills (`review-work`, `review-plan`, `review-commits`, `review-files`, `review-feature`) all terminate by invoking `ok-planner:review-cleanup`, which dispatches a fixer subagent and loops until the reviewer reports zero issues. Guard rail: max 3 fix-review cycles.

When editing review skills, preserve two invariants:
1. The reviewer output flows to `review-cleanup` **verbatim** — do not add filtering, categorization, or severity triage logic.
2. `review-cleanup` is marked internal (not user-facing); do not expose it as a slash command.

## Output styles vs. skills (where does content belong?)

- **Skills** = capabilities invoked by slash command (e.g., `/brainstorm`). On-demand, user-triggered.
- **Output styles** = standing behavioral rules that apply across every turn of a session. One active style at a time; the user picks via `/config` → Output style. Styles ship as `output-styles/<name>.md`; the filename becomes the style name unless frontmatter `name` overrides it.
- Custom output styles **replace** Claude Code's built-in coding instructions by default. Any style in this repo intended to add *mild* rules must set `keep-coding-instructions: true` in frontmatter so it layers on top of defaults rather than replacing them. `ok-conduct` is the canonical example.
- Rule of thumb: **rules that should always apply** go in an output style. **Workflows the user chooses to run** go in a skill.

## Adding a new skill

1. Create `skills/<skill>/SKILL.md` with the required frontmatter.
2. If the skill is a new review type, make it dispatch a reviewer and then invoke `ok-planner:review-cleanup` with the raw output.
3. If the skill should show up in the session-start context listing, add it to the table in `skills/ok-planner/SKILL.md` (the hook reads that file).
4. No manifest change is needed — skills are discovered from the `skills/` directory.

## Constraints specific to this repo

- Never commit `.claude/settings.local.json` — it is gitignored.
- Do not create `.ok-planner/specs/` or `.ok-planner/plans/` files in this repo unless you are dogfooding a skill — those paths are conventions the skills write into *consumer* projects, not artifacts that belong here.
- There are no `package.json` files and no JS build — skills are pure markdown and the one hook is bash. Do not add Node tooling unless a skill actually needs it.
