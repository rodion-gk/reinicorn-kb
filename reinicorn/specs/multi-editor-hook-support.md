---
type: spec
title: Multi-Editor Hook Support
slug: multi-editor-hook-support
lifecycle: done
status: implemented
created: 2026-02-25
author: mnbiehl
origin: ai-assisted
human_validated: false
implemented_by: feature-mvp
---

# Multi-Editor Hook Support

## Problem

The doc-template enforcement hook (`enforce-doc-templates.sh`) only works via Claude Code's `PreToolUse` hook system. Copilot and Cursor agents can bypass enforcement entirely. Project docs and skills position multi-editor compatibility, but actual enforcement is Claude-only.

## Design Goals

- One hook script serves Claude Code, VS Code Copilot, and Cursor
- Script lives at a neutral path (`.reins/hooks/`) not under any editor's config dir
- `hooks_install.py` writes all three editor config files
- Exit-code-only signaling (0 allow, 2 block) — universally supported
- Script handles all three stdin JSON formats via jq fallback chain

## Design

### Portable Hook Script (`.reins/hooks/enforce-doc-templates.sh`)

```bash
INPUT=$(cat)

# Tool name check (Copilot ignores matchers, fires for ALL tools)
TOOL=$(echo "$INPUT" | jq -r '.tool_name // empty')
case "$TOOL" in
  Write|Edit|write|edit|editFiles|file_edit) ;;
  *) exit 0 ;;
esac

# File path — all three editors use .tool_input:
#   Claude Code: .tool_input.file_path
#   Cursor:      .tool_input.file_path / .tool_input.filePath
#   Copilot:     .tool_input.file_path / .tool_input.files[0]
FILE=$(echo "$INPUT" | jq -r '
  .tool_input.file_path //
  .tool_input.filePath //
  (.tool_input.files // [])[0] //
  empty
')

[ -z "$FILE" ] && exit 0

case "$FILE" in
  */harness/*.md | */harness/*/*.md | */harness/*/*/*.md | */harness/*/*/*/*.md) ;;
  *) exit 0 ;;
esac

reins doc check-path "$FILE"
```

### Editor Config Files

**`.claude/settings.json`** — update command path from `.claude/hooks/` to `.reins/hooks/`:
```json
{"hooks": {"PreToolUse": [{"matcher": "Write|Edit", "hooks": [{"type": "command", "command": ".reins/hooks/enforce-doc-templates.sh"}]}]}}
```

**`.cursor/hooks.json`** (new):
```json
{"version": 1, "hooks": {"preToolUse": [{"command": ".reins/hooks/enforce-doc-templates.sh", "matcher": "Write|Edit"}]}}
```

**`.github/hooks/reins.json`** (new):
```json
{"version": 1, "hooks": {"preToolUse": [{"type": "command", "bash": ".reins/hooks/enforce-doc-templates.sh"}]}}
```

### `hooks_install.py` Changes

- Rename `_CLAUDE_HOOKS` → `_EDITOR_HOOKS` with per-editor config entries
- Rename `_install_claude_hooks()` → `_install_editor_hooks()`
- Add config writers for Cursor and Copilot alongside existing Claude writer
- Script source asset dir: `editor-hooks/` (was `claude-hooks/`)
- Script destination: `.reins/hooks/` (was `.claude/hooks/`)

### Migration

- Move `.claude/hooks/enforce-doc-templates.sh` → `.reins/hooks/enforce-doc-templates.sh`
- Update `.claude/settings.json` command path
- Remove old `.claude/hooks/` dir if empty

## Non-Goals

- JSON stdout responses (Copilot `permissionDecision` / Cursor `permission` fields) — exit codes suffice
- Per-editor wrapper scripts — jq fallback chain handles format differences
- Replacing the shell script with Python — avoids uv startup overhead on every tool call
