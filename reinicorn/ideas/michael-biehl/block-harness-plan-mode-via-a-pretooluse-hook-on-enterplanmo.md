---
type: idea
title: Block harness plan mode via a PreToolUse hook on EnterPlanMode - reinicorn owns
slug: block-harness-plan-mode-via-a-pretooluse-hook-on-enterplanmo
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
---

# Block harness plan mode via a PreToolUse hook on EnterPlanMode - reinicorn owns 

## Description

Block harness plan mode via a PreToolUse hook on EnterPlanMode - reinicorn owns the plan lane and Claude Code plan mode silently competes with it

## Notes

### What happened

An agent session on the frontmatter spec auto-transitioned into Claude Code plan
mode and wrote its implementation plan to `~/.claude/plans/`, not to the kb. The
plan lane reinicorn owns (`rcorn plan create` →
`exec-plans/active/<branch>/plan.md`) was silently bypassed. Nothing in the repo
resisted it: the guardrail hooks cover kb *writes*, not the competing lane.

This is new harness behavior — `EnterPlanMode` is a model-invocable tool, so it
fires on the agent's initiative rather than from a mode the user selected.

### Verified findings (Claude Code 2.1.220)

- **No setting disables plan mode.** `disableAutoMode` exists in the bundle (11
  string hits; documented as removing `auto` from the Shift+Tab cycle and
  rejecting `--permission-mode auto` at startup). `disablePlanMode` and
  `planModeDisabled`: zero hits. There is no knob to point adopters at.
- **`EnterPlanMode` is a real tool** — `var Vie="EnterPlanMode"`, whose `call()`
  flips the session mode.
- **A `PreToolUse` hook with matcher `EnterPlanMode` DOES intercept it.**
  Tested in-session: exit 2 blocked the call and fed stderr back to the agent.
  This is undocumented — the hooks reference does not enumerate the tool.
- **It is interactive-only.** Under `claude -p`, the tool reports "exists but is
  not enabled in this context", matching the bundle string "EnterPlanMode tool
  cannot be used in agent contexts". So the hook is a no-op headlessly, which is
  fine — headless runs cannot enter plan mode anyway.
- Settings picked the hook up without a session restart.

### Sketch

A third entry in `_EDITOR_HOOK_SCRIPTS` (`commands/hooks_install.py:19`),
directly parallel to `block-raw-kb-git.sh` — a harness-native capability that
competes with an `rcorn` lane, blocked at exit 2 with a message naming the right
command:

```
Blocked: plan mode competes with the reinicorn plan lane.

Plans belong in the kb:
  uv run rcorn plan create     — create the exec plan for this branch
  uv run rcorn plan show       — read it back
```

### Open questions

- **Cross-editor contract.** This would be the first Claude-Code-only entry in
  `_EDITOR_HOOK_SCRIPTS`. Cursor and Copilot have no plan mode, and
  `_copilot_entry` discards the matcher outright, so it would install a no-op
  into Copilot config. Needs a per-editor gate.
- **Block or redirect?** Hard-blocking `EnterPlanMode` also blocks legitimate
  pre-implementation exploration. An alternative is to allow it but have the
  hook inject a reminder that the plan must land via `rcorn plan create` —
  weaker, but it does not fight the harness.
- **Undocumented surface.** The interception is not in the hooks reference, so
  it could change without notice. Worth a test that fails loudly if a Claude
  Code upgrade stops honoring the matcher.
- **Instruction-level backstop.** `templates/AGENTS.md` and
  `platform-instructions/` should state that plans live in the kb regardless,
  so non-Claude-Code harnesses get the norm even without enforcement.
