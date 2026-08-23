---
type: plan
title: Cross-Platform Support Implementation Plan
slug: feature-cross-platform-support
lifecycle: dropped
status: abandoned
created: 2026-07-17
author: Michael Biehl
branch: feature-cross-platform-support
---

# Cross-Platform Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add first-class support for Cursor, Copilot, Gemini, and Codex to `reins` by adopting cross-tool patterns from `obra/superpowers` (MIT) — tool-name translation references, a polyglot SessionStart bootstrap hook, and plugin manifests for marketplace distribution.

**Architecture:** Three layers, one PR.
1. **Tool-name translation** — new `.claude/skills/using-reins/references/{codex,copilot,gemini}-tools.md` map Claude Code tool names used inside skill bodies (`Read`, `Write`, `Edit`, `Task`, `TodoWrite`, `Skill`) to each platform's equivalents.
2. **Polyglot SessionStart bootstrap** — new `hooks/session-start-bootstrap` (bash) + `hooks/run-hook.cmd` (Windows polyglot wrapper) detect platform via env vars (`CLAUDE_PLUGIN_ROOT`, `CURSOR_PLUGIN_ROOT`, `COPILOT_CLI`, etc.) and emit `using-reins` skill content in the JSON shape each platform expects. `reins hooks install` wires it into each platform's config alongside the existing PreToolUse hooks.
3. **Plugin manifests** — `.claude-plugin/plugin.json` + `marketplace.json`, `.cursor-plugin/plugin.json`, `gemini-extension.json`, `.codex/INSTALL.md` at repo root so `reins` is distributable through each platform's plugin/extension channel.

Attribution: `LICENSE-THIRD-PARTY.md` at root listing the MIT-licensed upstream (obra/superpowers), one-line provenance comment on files copied essentially verbatim, and a Credits section in README.

**Out of scope (explicitly deferred):**
- OpenCode JS plugin (`.opencode/plugins/reins.js`) — noted in `decisions.md` as v2.
- Changing existing `.claude/hooks/session-start.sh` behavior beyond extracting the kb-autoupdate block (the skill-injection job is a *separate* new script, not a replacement).

**Tech Stack:** Python 3.12+ (reins CLI, pytest), bash (hooks), JSON (manifests), Markdown (skill content). Assets bundled for wheel install via `pyproject.toml` `[tool.hatch.build.targets.wheel.force-include]` — `hooks/` is already listed, so new files in there ship automatically.

---

## File Structure

### New files
- `LICENSE-THIRD-PARTY.md` — attribution for obra/superpowers MIT content
- `.claude/skills/using-reins/references/codex-tools.md`
- `.claude/skills/using-reins/references/copilot-tools.md`
- `.claude/skills/using-reins/references/gemini-tools.md`
- `hooks/session-start-bootstrap` — polyglot SessionStart script
- `hooks/run-hook.cmd` — Windows batch wrapper
- `hooks/hooks-cursor-session-start.json` — cursor-format config snippet (for documentation/reference)
- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `.cursor-plugin/plugin.json`
- `gemini-extension.json`
- `GEMINI.md` — Gemini extension context-file entry point
- `.codex/INSTALL.md`
- `tests/hooks/test_session_start_bootstrap.py` — platform-detection JSON shape tests
- `tests/test_hooks_install_session_start.py` — config-file merge tests
- `tests/test_plugin_manifests.py` — manifest JSON validates
- `tests/test_platform_references.py` — tool-reference files exist + rename complete

### Modified files
- `.claude/skills/using-reins/SKILL.md` — add "Platform Adaptation" section pointing at refs
- `src/reins/commands/hooks_install.py` — add SessionStart entry per platform
- `README.md` — add Credits section + per-platform install instructions
- `platform-instructions/{claude,copilot,cursor,gemini}.md` — note that SessionStart bootstrap is installed by `reins hooks install`

---

## Phase 0 — Attribution scaffolding

### Task 0.1: Create `LICENSE-THIRD-PARTY.md`

**Files:**
- Create: `LICENSE-THIRD-PARTY.md`

- [ ] **Step 1: Fetch upstream LICENSE**

```bash
gh api repos/obra/superpowers/contents/LICENSE --jq '.content' | base64 -d > /tmp/obra-LICENSE
cat /tmp/obra-LICENSE
```

Expected: MIT license text © Jesse Vincent.

- [ ] **Step 2: Write `LICENSE-THIRD-PARTY.md`**

```markdown
# Third-Party Licenses

This project includes content derived from other open-source projects.
Where content is copied essentially verbatim, the source file carries a
one-line provenance comment. This document records the full upstream
license texts.

## obra/superpowers

Portions of `.claude/skills/using-reins/references/`, `hooks/session-start-bootstrap`,
and `hooks/run-hook.cmd` are adapted from the superpowers skills framework:
https://github.com/obra/superpowers (MIT License)

<PASTE FULL TEXT OF /tmp/obra-LICENSE HERE>
```

Replace the `<PASTE ...>` line with the contents of `/tmp/obra-LICENSE`.

- [ ] **Step 3: Commit**

```bash
git add LICENSE-THIRD-PARTY.md
git commit -m "docs: add LICENSE-THIRD-PARTY.md for upstream MIT content"
```

---

## Phase 1 — Tool-name translation references

### Task 1.1: Failing test for references directory

**Files:**
- Create: `tests/test_platform_references.py`

- [ ] **Step 1: Write the failing test**

```python
"""Platform tool-name reference files exist and reference 'reins', not upstream 'superpowers'."""
from __future__ import annotations

from pathlib import Path

REFS_DIR = Path(__file__).resolve().parents[1] / ".claude" / "skills" / "using-reins" / "references"
EXPECTED = ["codex-tools.md", "copilot-tools.md", "gemini-tools.md"]

def test_all_reference_files_exist():
    for name in EXPECTED:
        path = REFS_DIR / name
        assert path.is_file(), f"{path} missing"

def test_no_upstream_branding_in_references():
    for name in EXPECTED:
        content = (REFS_DIR / name).read_text().lower()
        assert "superpowers" not in content, f"{name} still mentions 'superpowers'"
        assert "using-superpowers" not in content
```

- [ ] **Step 2: Run — expect FAIL**

```bash
uv run pytest tests/test_platform_references.py -v
```

Expected: FAIL ("REFS_DIR missing" or similar).

### Task 1.2: Create `codex-tools.md`

**Files:**
- Create: `.claude/skills/using-reins/references/codex-tools.md`

- [ ] **Step 1: Fetch upstream and adapt**

```bash
mkdir -p .claude/skills/using-reins/references
gh api repos/obra/superpowers/contents/skills/using-superpowers/references/codex-tools.md \
  --jq '.content' | base64 -d > .claude/skills/using-reins/references/codex-tools.md
```

- [ ] **Step 2: Rename `superpowers` → `reins` and add provenance header**

Edit the file. Replace every `superpowers` with `reins` and every `using-superpowers` with `using-reins`. Add this as line 1 (before the existing content):

```markdown
<!-- Derived from obra/superpowers (MIT) — see LICENSE-THIRD-PARTY.md -->
```

- [ ] **Step 3: Verify "superpowers" is gone**

```bash
grep -i superpowers .claude/skills/using-reins/references/codex-tools.md && echo "STILL PRESENT — FIX"
```

Expected: empty output (no matches).

### Task 1.3: Create `copilot-tools.md`

**Files:**
- Create: `.claude/skills/using-reins/references/copilot-tools.md`

- [ ] **Step 1: Fetch + adapt**

```bash
gh api repos/obra/superpowers/contents/skills/using-superpowers/references/copilot-tools.md \
  --jq '.content' | base64 -d > .claude/skills/using-reins/references/copilot-tools.md
```

- [ ] **Step 2: Same rename + provenance header as Task 1.2 Step 2**

- [ ] **Step 3: Verify**

```bash
grep -i superpowers .claude/skills/using-reins/references/copilot-tools.md && echo "STILL PRESENT — FIX"
```

Expected: empty.

### Task 1.4: Create `gemini-tools.md`

**Files:**
- Create: `.claude/skills/using-reins/references/gemini-tools.md`

- [ ] **Step 1: Fetch + adapt**

```bash
gh api repos/obra/superpowers/contents/skills/using-superpowers/references/gemini-tools.md \
  --jq '.content' | base64 -d > .claude/skills/using-reins/references/gemini-tools.md
```

- [ ] **Step 2: Same rename + provenance header**

- [ ] **Step 3: Verify**

```bash
grep -i superpowers .claude/skills/using-reins/references/gemini-tools.md && echo "STILL PRESENT — FIX"
```

Expected: empty.

### Task 1.5: Wire references into `using-reins/SKILL.md`

**Files:**
- Modify: `.claude/skills/using-reins/SKILL.md`

- [ ] **Step 1: Add "Platform Adaptation" section**

After the existing `## How to Access Skills` section in `.claude/skills/using-reins/SKILL.md`, insert:

```markdown
## Platform Adaptation

Skills in this project use Claude Code tool names (`Read`, `Write`, `Edit`, `Task`, `TodoWrite`, `Skill`, `Bash`, `Grep`, `Glob`). If you are running under a different platform, use the equivalent tool from your environment. Mappings:

- Codex: see `references/codex-tools.md`
- GitHub Copilot CLI: see `references/copilot-tools.md`
- Gemini CLI: see `references/gemini-tools.md`

(Gemini CLI users: your context file `GEMINI.md` auto-includes `gemini-tools.md`, so no manual load is needed.)
```

- [ ] **Step 2: Run the test from Task 1.1 — expect PASS**

```bash
uv run pytest tests/test_platform_references.py -v
```

Expected: PASS.

### Task 1.6: Commit Phase 1

- [ ] **Step 1: Commit**

```bash
git add .claude/skills/using-reins/references tests/test_platform_references.py \
        .claude/skills/using-reins/SKILL.md
git commit -m "feat: add cross-platform tool-name references for using-reins skill

Derived from obra/superpowers (MIT). See LICENSE-THIRD-PARTY.md."
```

---

## Phase 2 — Polyglot SessionStart bootstrap hook

### Task 2.1: Record session-start audit in decisions.md

**Files:**
- Modify: `kb/reins/exec-plans/active/feature-cross-platform-support/decisions.md`

- [ ] **Step 1: Append to decisions.md**

```markdown
## Session-start script split

The existing `.claude/hooks/session-start.sh` does two things:
1. **Always:** pull kb submodule to avoid divergence (auto-update for all users of a reins-managed project).
2. **Remote-only (`CLAUDE_CODE_REMOTE=true`):** rewrite git credentials for private submodules, `uv sync` dev deps, install editable reins CLI, run `reins hooks install`.

Block (2) is contributor-specific. Block (1) is general auto-update
infrastructure that all end-user projects should have.

**Decision:** This plan does NOT modify session-start.sh. It adds a separate
new script `hooks/session-start-bootstrap` whose sole job is skill-content
injection into the agent's context. The two scripts coexist: session-start.sh
handles environment (kb pull, optional dev setup) and session-start-bootstrap
handles behavior (inject using-reins skill content). They are wired into the
platform's SessionStart hook chain independently via reins hooks install.

**Followup (not in this plan):** make kb auto-update togglable via
`.reins-config` (e.g. `REINS_AUTO_SYNC_ON_SESSION=false`). Tracked separately.
```

- [ ] **Step 2: Commit**

```bash
git add kb/reins/exec-plans/active/feature-cross-platform-support/decisions.md
git commit -m "docs(plan): record session-start split decision"
```

### Task 2.2: Write failing test — bootstrap script exists and is executable

**Files:**
- Create: `tests/hooks/test_session_start_bootstrap.py`

- [ ] **Step 1: Write the test**

```python
"""Tests for the polyglot hooks/session-start-bootstrap script."""
from __future__ import annotations

import json
import os
import subprocess
from pathlib import Path

REPO_ROOT = Path(__file__).resolve().parents[2]
SCRIPT = REPO_ROOT / "hooks" / "session-start-bootstrap"

def test_script_exists_and_is_executable():
    assert SCRIPT.is_file(), f"{SCRIPT} missing"
    assert os.access(SCRIPT, os.X_OK), f"{SCRIPT} not executable"
```

- [ ] **Step 2: Ensure `tests/hooks/__init__.py` exists**

```bash
mkdir -p tests/hooks
touch tests/hooks/__init__.py
```

- [ ] **Step 3: Run — expect FAIL**

```bash
uv run pytest tests/hooks/test_session_start_bootstrap.py -v
```

Expected: FAIL (script missing).

### Task 2.3: Create `hooks/session-start-bootstrap`

**Files:**
- Create: `hooks/session-start-bootstrap`

- [ ] **Step 1: Write the polyglot script**

```bash
cat > hooks/session-start-bootstrap <<'EOF'
#!/usr/bin/env bash
# Derived from obra/superpowers (MIT) — see LICENSE-THIRD-PARTY.md
# SessionStart bootstrap: inject using-reins skill content into the agent's
# context in the JSON shape each host platform expects.

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPO_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

SKILL_PATH="${REPO_ROOT}/.claude/skills/using-reins/SKILL.md"
skill_content=""
if [ -f "$SKILL_PATH" ]; then
  skill_content=$(cat "$SKILL_PATH")
fi

escape_for_json() {
  local s="$1"
  s="${s//\\/\\\\}"
  s="${s//\"/\\\"}"
  s="${s//$'\n'/\\n}"
  s="${s//$'\r'/\\r}"
  s="${s//$'\t'/\\t}"
  printf '%s' "$s"
}

skill_escaped=$(escape_for_json "$skill_content")
context="<EXTREMELY_IMPORTANT>\nYou have reins skills available. Below is the full content of your 'using-reins' skill — follow it for any task.\n\n${skill_escaped}\n</EXTREMELY_IMPORTANT>"

# Platform detection. Env vars:
#   CURSOR_PLUGIN_ROOT  — Cursor
#   CLAUDE_PLUGIN_ROOT  — Claude Code (unless also COPILOT_CLI)
#   COPILOT_CLI         — Copilot CLI
#   default             — Codex / Gemini / unknown — use SDK-standard shape
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  printf '{\n  "additional_context": "%s"\n}\n' "$context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$context"
else
  printf '{\n  "additionalContext": "%s"\n}\n' "$context"
fi

exit 0
EOF
chmod +x hooks/session-start-bootstrap
```

- [ ] **Step 2: Run test from 2.2 — expect PASS**

```bash
uv run pytest tests/hooks/test_session_start_bootstrap.py -v
```

Expected: PASS.

### Task 2.4: Test Claude Code JSON shape

**Files:**
- Modify: `tests/hooks/test_session_start_bootstrap.py`

- [ ] **Step 1: Append test**

```python
def test_claude_code_shape():
    env = {**os.environ, "CLAUDE_PLUGIN_ROOT": "/fake"}
    env.pop("CURSOR_PLUGIN_ROOT", None)
    env.pop("COPILOT_CLI", None)
    result = subprocess.run([str(SCRIPT)], capture_output=True, text=True, env=env, check=True)
    parsed = json.loads(result.stdout)
    assert "hookSpecificOutput" in parsed
    assert parsed["hookSpecificOutput"]["hookEventName"] == "SessionStart"
    assert "additionalContext" in parsed["hookSpecificOutput"]
    assert "using-reins" in parsed["hookSpecificOutput"]["additionalContext"]
```

- [ ] **Step 2: Run — expect PASS**

```bash
uv run pytest tests/hooks/test_session_start_bootstrap.py::test_claude_code_shape -v
```

Expected: PASS.

### Task 2.5: Test Cursor JSON shape

- [ ] **Step 1: Append test**

```python
def test_cursor_shape():
    env = {**os.environ, "CURSOR_PLUGIN_ROOT": "/fake"}
    env.pop("COPILOT_CLI", None)
    result = subprocess.run([str(SCRIPT)], capture_output=True, text=True, env=env, check=True)
    parsed = json.loads(result.stdout)
    assert "additional_context" in parsed
    assert "using-reins" in parsed["additional_context"]
    assert "hookSpecificOutput" not in parsed
```

- [ ] **Step 2: Run — expect PASS**

```bash
uv run pytest tests/hooks/test_session_start_bootstrap.py::test_cursor_shape -v
```

Expected: PASS.

### Task 2.6: Test Copilot/default JSON shape

- [ ] **Step 1: Append test**

```python
def test_copilot_shape():
    env = {**os.environ, "COPILOT_CLI": "1"}
    env.pop("CURSOR_PLUGIN_ROOT", None)
    env.pop("CLAUDE_PLUGIN_ROOT", None)
    result = subprocess.run([str(SCRIPT)], capture_output=True, text=True, env=env, check=True)
    parsed = json.loads(result.stdout)
    assert "additionalContext" in parsed
    assert "hookSpecificOutput" not in parsed
    assert "additional_context" not in parsed

def test_default_shape_when_no_env_set():
    env = {k: v for k, v in os.environ.items()
           if k not in ("CURSOR_PLUGIN_ROOT", "CLAUDE_PLUGIN_ROOT", "COPILOT_CLI")}
    result = subprocess.run([str(SCRIPT)], capture_output=True, text=True, env=env, check=True)
    parsed = json.loads(result.stdout)
    assert "additionalContext" in parsed
```

- [ ] **Step 2: Run all bootstrap tests — expect PASS**

```bash
uv run pytest tests/hooks/test_session_start_bootstrap.py -v
```

Expected: 5 passed.

### Task 2.7: Create `hooks/run-hook.cmd` Windows wrapper

**Files:**
- Create: `hooks/run-hook.cmd`

- [ ] **Step 1: Fetch upstream, adapt header**

```bash
gh api repos/obra/superpowers/contents/hooks/run-hook.cmd --jq '.content' | base64 -d > hooks/run-hook.cmd
```

- [ ] **Step 2: Add provenance header on line 1**

Prepend this line to the file:

```
: << 'PROVENANCE'
Derived from obra/superpowers (MIT) — see LICENSE-THIRD-PARTY.md
PROVENANCE
```

(Matches the existing polyglot `: << 'CMDBLOCK'` style; bash sees as no-op, cmd ignores.)

The file is polyglot — no per-line renames required (no `superpowers` strings inside).

- [ ] **Step 3: Sanity check — verify the wrapper can invoke our bootstrap on Unix**

```bash
chmod +x hooks/run-hook.cmd
./hooks/run-hook.cmd session-start-bootstrap | head -3
```

Expected: same JSON output as direct invocation.

### Task 2.8: Commit Phase 2

- [ ] **Step 1: Commit**

```bash
git add hooks/session-start-bootstrap hooks/run-hook.cmd tests/hooks/
git commit -m "feat: add polyglot SessionStart bootstrap hook

hooks/session-start-bootstrap detects the host platform via env vars
(CLAUDE_PLUGIN_ROOT / CURSOR_PLUGIN_ROOT / COPILOT_CLI) and emits the
using-reins skill content in the JSON shape each platform expects.
hooks/run-hook.cmd is a Windows polyglot wrapper so bash hooks run under
cmd.exe.

Derived from obra/superpowers (MIT). See LICENSE-THIRD-PARTY.md."
```

---

## Phase 3 — Wire SessionStart into `reins hooks install`

### Task 3.1: Failing test — Claude settings get SessionStart entry

**Files:**
- Create: `tests/test_hooks_install_session_start.py`

- [ ] **Step 1: Write the test**

```python
"""hooks install should register SessionStart bootstrap in each platform's config."""
from __future__ import annotations

import json
import subprocess
from pathlib import Path

from reins.commands.hooks_install import cmd_hooks_install

def _init_repo(path: Path) -> None:
    path.mkdir(parents=True, exist_ok=True)
    subprocess.run(["git", "init", "-q"], cwd=path, check=True)
    subprocess.run(["git", "config", "user.email", "t@t"], cwd=path, check=True)
    subprocess.run(["git", "config", "user.name", "t"], cwd=path, check=True)
    subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=path, check=True)

def test_claude_settings_gets_session_start(tmp_path: Path, monkeypatch):
    repo = tmp_path / "r"
    _init_repo(repo)
    monkeypatch.chdir(repo)
    rc = cmd_hooks_install()
    assert rc == 0
    settings = json.loads((repo / ".claude" / "settings.json").read_text())
    ss = settings.get("hooks", {}).get("SessionStart", [])
    assert ss, "no SessionStart entry"
    cmd_field = ss[0].get("hooks", [{}])[0].get("command", "")
    assert "session-start-bootstrap" in cmd_field
```

- [ ] **Step 2: Run — expect FAIL**

```bash
uv run pytest tests/test_hooks_install_session_start.py::test_claude_settings_gets_session_start -v
```

Expected: FAIL (no SessionStart installed).

### Task 3.2: Implement Claude SessionStart merge

**Files:**
- Modify: `src/reins/commands/hooks_install.py`

- [ ] **Step 1: Add constant near other hook-entry constants (line ~19)**

After the existing `_COPILOT_ENTRY` definition, add:

```python
_BOOTSTRAP_SCRIPT = "hooks/session-start-bootstrap"
_CLAUDE_SS_ENTRY = {
    "hooks": [{"type": "command", "command": f"$CLAUDE_PROJECT_DIR/{_BOOTSTRAP_SCRIPT}"}],
}
_CURSOR_SS_ENTRY = {"command": f"./{_BOOTSTRAP_SCRIPT}"}
_COPILOT_SS_ENTRY = {"type": "command", "bash": f"./{_BOOTSTRAP_SCRIPT}"}
```

- [ ] **Step 2: Extend `_merge_claude_settings` to accept an event name**

Change the signature and key lookup to:

```python
def _merge_claude_settings(
    settings_path: Path,
    hook_entries: list[dict],
    event: str = "PreToolUse",
    dedupe_key: str = "matcher",
) -> None:
    # ... existing load logic ...
    hooks = settings.get("hooks")
    if not isinstance(hooks, dict):
        hooks = {}
        settings["hooks"] = hooks
    bucket = hooks.get(event)
    if not isinstance(bucket, list):
        bucket = []
        hooks[event] = bucket
    existing = {e.get(dedupe_key) for e in bucket if isinstance(e, dict)}
    added = 0
    for entry in hook_entries:
        if entry.get(dedupe_key) not in existing:
            bucket.append(entry)
            added += 1
    # ... existing write + log logic ...
```

Update the two callers (cursor, copilot merge) similarly — pass the event name (`"PreToolUse"` default for existing calls; `"SessionStart"` / `"sessionStart"` for new ones).

Dedupe key: SessionStart entries lack a `matcher`, so use the nested command string. Simplest: dedupe by `json.dumps(entry, sort_keys=True)` across the bucket.

Replace the dedupe logic with this version that works for both events:

```python
existing = {json.dumps(e, sort_keys=True) for e in bucket if isinstance(e, dict)}
added = 0
for entry in hook_entries:
    if json.dumps(entry, sort_keys=True) not in existing:
        bucket.append(entry)
        added += 1
```

- [ ] **Step 3: Call the new SessionStart merges in `_install_editor_hooks`**

After the existing three `_merge_*` calls, add:

```python
_merge_claude_settings(root / ".claude" / "settings.json", [_CLAUDE_SS_ENTRY], event="SessionStart")
_merge_cursor_settings(root / ".cursor" / "hooks.json", [_CURSOR_SS_ENTRY], event="sessionStart")
_merge_copilot_settings(root / ".github" / "hooks" / "reins.json", [_COPILOT_SS_ENTRY], event="sessionStart")
```

- [ ] **Step 4: Apply the same `event` parameter + dedupe-by-json change to `_merge_cursor_settings` and `_merge_copilot_settings`**

Each function: add `event: str = "preToolUse"` kwarg, look up `hooks[event]` instead of hard-coded key, switch dedupe to `json.dumps`.

- [ ] **Step 5: Run full hooks-install test suite — expect PASS**

```bash
uv run pytest tests/ -k "hooks_install" -v
```

Expected: existing tests still pass (they used hardcoded keys but dedupe by `json.dumps` still matches unique entries) + the new Task 3.1 test passes.

If any existing test fails, it's because it asserted on `hooks["PreToolUse"]` specifically — confirm that's still where PreToolUse goes and adjust only if our refactor broke the key name (it shouldn't, since the default kwarg matches).

### Task 3.3: Failing test — Cursor SessionStart

- [ ] **Step 1: Append to `tests/test_hooks_install_session_start.py`**

```python
def test_cursor_hooks_gets_session_start(tmp_path: Path, monkeypatch):
    repo = tmp_path / "r"
    _init_repo(repo)
    monkeypatch.chdir(repo)
    rc = cmd_hooks_install()
    assert rc == 0
    settings = json.loads((repo / ".cursor" / "hooks.json").read_text())
    ss = settings.get("hooks", {}).get("sessionStart", [])
    assert ss
    assert "session-start-bootstrap" in ss[0].get("command", "")
```

- [ ] **Step 2: Run — expect PASS (if Task 3.2 Step 3 is complete)**

```bash
uv run pytest tests/test_hooks_install_session_start.py::test_cursor_hooks_gets_session_start -v
```

Expected: PASS.

### Task 3.4: Failing test — Copilot SessionStart

- [ ] **Step 1: Append**

```python
def test_copilot_hooks_gets_session_start(tmp_path: Path, monkeypatch):
    repo = tmp_path / "r"
    _init_repo(repo)
    monkeypatch.chdir(repo)
    rc = cmd_hooks_install()
    assert rc == 0
    settings = json.loads((repo / ".github" / "hooks" / "reins.json").read_text())
    ss = settings.get("hooks", {}).get("sessionStart", [])
    assert ss
    assert "session-start-bootstrap" in ss[0].get("bash", "")
```

- [ ] **Step 2: Run — expect PASS**

```bash
uv run pytest tests/test_hooks_install_session_start.py -v
```

Expected: all 3 pass.

### Task 3.5: Commit Phase 3

- [ ] **Step 1: Commit**

```bash
git add src/reins/commands/hooks_install.py tests/test_hooks_install_session_start.py
git commit -m "feat(hooks): install SessionStart bootstrap for all editor platforms

reins hooks install now registers hooks/session-start-bootstrap as the
SessionStart hook in .claude/settings.json, .cursor/hooks.json, and
.github/hooks/reins.json alongside the existing PreToolUse entries."
```

---

## Phase 4 — Plugin manifests (reins-as-plugin)

### Task 4.1: Failing test — each manifest validates

**Files:**
- Create: `tests/test_plugin_manifests.py`

- [ ] **Step 1: Write**

```python
"""Plugin/extension manifests at repo root validate for each host platform."""
from __future__ import annotations

import json
from pathlib import Path

REPO_ROOT = Path(__file__).resolve().parents[1]

def _load(rel: str) -> dict:
    p = REPO_ROOT / rel
    assert p.is_file(), f"{rel} missing"
    return json.loads(p.read_text())

def test_claude_plugin_manifest():
    m = _load(".claude-plugin/plugin.json")
    assert m["name"] == "reins"
    assert m["license"] == "MIT"
    assert "version" in m

def test_claude_marketplace_manifest():
    m = _load(".claude-plugin/marketplace.json")
    assert m["name"] == "reins-dev"
    assert m["plugins"][0]["name"] == "reins"

def test_cursor_plugin_manifest():
    m = _load(".cursor-plugin/plugin.json")
    assert m["name"] == "reins"
    assert m["skills"].endswith("/skills/") or m["skills"].endswith("/skills")

def test_gemini_extension_manifest():
    m = _load("gemini-extension.json")
    assert m["name"] == "reins"
    assert m["contextFileName"] == "GEMINI.md"

def test_codex_install_docs_exist():
    assert (REPO_ROOT / ".codex" / "INSTALL.md").is_file()

def test_gemini_context_file_exists():
    assert (REPO_ROOT / "GEMINI.md").is_file()
```

- [ ] **Step 2: Run — expect FAIL (6 failures)**

```bash
uv run pytest tests/test_plugin_manifests.py -v
```

### Task 4.2: Write `.claude-plugin/`

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Resolve the version from pyproject.toml**

```bash
grep -E '^version = ' pyproject.toml
```

Note the version string for the next step (e.g. `0.1.0`).

- [ ] **Step 2: Write `plugin.json`**

```json
{
  "name": "reins",
  "description": "Knowledge-base + skills harness for agentic coding across Claude Code, Cursor, Copilot, Gemini, and Codex",
  "version": "<VERSION-FROM-PYPROJECT>",
  "author": {
    "name": "Michael Biehl",
    "email": "you@example.com"
  },
  "homepage": "https://github.com/mnbiehl/reins",
  "repository": "https://github.com/mnbiehl/reins",
  "license": "MIT",
  "keywords": ["skills", "kb", "knowledge-base", "agents", "cross-platform"]
}
```

- [ ] **Step 3: Write `marketplace.json`**

```json
{
  "name": "reins-dev",
  "description": "Development marketplace for reins",
  "owner": {
    "name": "Michael Biehl",
    "email": "you@example.com"
  },
  "plugins": [
    {
      "name": "reins",
      "description": "Knowledge-base + skills harness for agentic coding",
      "version": "<VERSION-FROM-PYPROJECT>",
      "source": "./",
      "author": {
        "name": "Michael Biehl",
        "email": "you@example.com"
      }
    }
  ]
}
```

Replace `<VERSION-FROM-PYPROJECT>` with the real version string.

### Task 4.3: Write `.cursor-plugin/plugin.json`

**Files:**
- Create: `.cursor-plugin/plugin.json`

- [ ] **Step 1: Write**

```json
{
  "name": "reins",
  "displayName": "Reins",
  "description": "Knowledge-base + skills harness for agentic coding",
  "version": "<VERSION-FROM-PYPROJECT>",
  "author": {
    "name": "Michael Biehl",
    "email": "you@example.com"
  },
  "homepage": "https://github.com/mnbiehl/reins",
  "repository": "https://github.com/mnbiehl/reins",
  "license": "MIT",
  "keywords": ["skills", "kb", "agents"],
  "skills": "./.claude/skills/",
  "hooks": "./hooks/hooks-cursor.json"
}
```

- [ ] **Step 2: Also create `hooks/hooks-cursor.json`**

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "command": "./hooks/session-start-bootstrap"
      }
    ]
  }
}
```

### Task 4.4: Write `gemini-extension.json` + `GEMINI.md`

**Files:**
- Create: `gemini-extension.json`
- Create: `GEMINI.md`

- [ ] **Step 1: Write `gemini-extension.json`**

```json
{
  "name": "reins",
  "description": "Knowledge-base + skills harness for agentic coding",
  "version": "<VERSION-FROM-PYPROJECT>",
  "contextFileName": "GEMINI.md"
}
```

- [ ] **Step 2: Write `GEMINI.md`**

```markdown
@./.claude/skills/using-reins/SKILL.md
@./.claude/skills/using-reins/references/gemini-tools.md
```

### Task 4.5: Write `.codex/INSTALL.md`

**Files:**
- Create: `.codex/INSTALL.md`

- [ ] **Step 1: Write**

```markdown
# Installing Reins for Codex

Codex discovers skills via `~/.agents/skills/`. Install reins by symlink.

## Prerequisites

- Git
- Codex CLI installed

## Installation

1. **Clone the reins repository:**
   ```bash
   git clone https://github.com/mnbiehl/reins.git ~/.codex/reins
   ```

2. **Create the skills symlink:**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/reins/.claude/skills ~/.agents/skills/reins
   ```

   **Windows (PowerShell):**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\reins" "$env:USERPROFILE\.codex\reins\.claude\skills"
   ```

3. **Restart Codex.**

## Verify

```bash
ls -la ~/.agents/skills/reins
```

You should see a symlink pointing at `~/.codex/reins/.claude/skills`.

## Tool mapping

Skills use Claude Code tool names. See
`~/.agents/skills/reins/using-reins/references/codex-tools.md` for the
full mapping.

## Updating

```bash
cd ~/.codex/reins && git pull
```

## Attribution

Install flow derived from obra/superpowers (MIT). See `LICENSE-THIRD-PARTY.md`.
```

### Task 4.6: Run Phase 4 tests and commit

- [ ] **Step 1: Run**

```bash
uv run pytest tests/test_plugin_manifests.py -v
```

Expected: all 6 pass.

- [ ] **Step 2: Commit**

```bash
git add .claude-plugin .cursor-plugin gemini-extension.json GEMINI.md .codex/ \
        hooks/hooks-cursor.json tests/test_plugin_manifests.py
git commit -m "feat: plugin manifests for Claude Code, Cursor, Gemini, Codex marketplaces

Makes reins installable from each host platform's plugin/extension system.
OpenCode JS plugin deferred to v2 (see exec-plans decisions.md)."
```

---

## Phase 5 — Wire platform-instructions templates

### Task 5.1: Update `platform-instructions/claude.md` to mention bootstrap

**Files:**
- Modify: `platform-instructions/claude.md`

- [ ] **Step 1: After the existing `## Skill Invocation` section, append:**

```markdown
## Session Bootstrap

On session start, the `reins hooks install` step (run once during `reins init`)
registers `hooks/session-start-bootstrap` as a SessionStart hook. This loads
the `using-reins` skill content into your context automatically at every
session start. No manual action required.
```

- [ ] **Step 2: Do the same for `platform-instructions/cursor.md`, `copilot.md`, `gemini.md`** — identical append.

- [ ] **Step 3: Verify templates still pass `tests/test_init_platforms.py`**

```bash
uv run pytest tests/test_init_platforms.py -v
```

Expected: 4 pass.

### Task 5.2: End-to-end init test — all platforms get SessionStart

**Files:**
- Modify: `tests/test_init_platforms.py`

- [ ] **Step 1: Append test**

```python
def test_init_installs_session_start_bootstrap_per_platform(tmp_path: Path):
    """`reins init` with all platforms selected installs SessionStart in every config."""
    repo = tmp_path / "my-repo"
    _init_repo(repo)

    with patch("reins.commands.init.repo_slug", return_value="my-repo"), \
         patch("reins.commands.init._prompt_platforms",
               return_value=["claude", "cursor", "copilot", "gemini"]):
        rc = cmd_init(kb_url="unused", local=True, cwd=repo)

    assert rc == 0
    claude = json.loads((repo / ".claude" / "settings.json").read_text())
    assert claude["hooks"].get("SessionStart")
    cursor = json.loads((repo / ".cursor" / "hooks.json").read_text())
    assert cursor["hooks"].get("sessionStart")
    copilot = json.loads((repo / ".github" / "hooks" / "reins.json").read_text())
    assert copilot["hooks"].get("sessionStart")
```

(Note: this test removes the `cmd_hooks_install` mock that earlier tests used, so `reins hooks install` actually runs. Ensure `tests/test_init_platforms.py` imports `json`.)

- [ ] **Step 2: Run — expect PASS**

```bash
uv run pytest tests/test_init_platforms.py::test_init_installs_session_start_bootstrap_per_platform -v
```

Expected: PASS (if Phases 2–3 are correct).

If it fails because `hooks/session-start-bootstrap` can't be found in the test tmp dir, audit `_find_asset_dir("hooks")` path resolution — the bootstrap needs to be copied into the target project as part of `cmd_hooks_install` or shipped in `reins/_data/hooks/`. Since `pyproject.toml` already force-includes `hooks/`, wheel installs work; editable installs work via `reins_root()`. Confirm the copy happens.

If the bootstrap is NOT currently copied into target projects by `cmd_hooks_install` (it only installs git hooks + editor pretooluse scripts), add the copy: the relevant editor-hooks copy block copies `enforce-doc-templates.sh` to `.reins/hooks/`. Add an analogous block that copies `session-start-bootstrap` + `run-hook.cmd` to `hooks/` in the target repo:

```python
# In _install_editor_hooks, after the existing enforce-doc-templates copy:
bootstrap_src = _find_asset_dir("hooks")
if bootstrap_src is not None:
    ss_src = bootstrap_src / "session-start-bootstrap"
    rh_src = bootstrap_src / "run-hook.cmd"
    target_hooks = root / "hooks"
    target_hooks.mkdir(parents=True, exist_ok=True)
    for src in (ss_src, rh_src):
        if src.is_file():
            dest = target_hooks / src.name
            if not dest.is_file() or dest.read_text() != src.read_text():
                shutil.copy2(src, dest)
                dest.chmod(dest.stat().st_mode | stat.S_IEXEC)
                console.success(f"INSTALLED: {src.name}")
```

Re-run the test. Expected: PASS.

### Task 5.3: Commit Phase 5

- [ ] **Step 1: Commit**

```bash
git add platform-instructions/ tests/test_init_platforms.py src/reins/commands/hooks_install.py
git commit -m "feat(init): install session-start-bootstrap into target projects

reins init now copies hooks/session-start-bootstrap and hooks/run-hook.cmd
into end-user project hooks/ dirs and registers them as SessionStart hooks
in each selected platform's config."
```

---

## Phase 6 — README + final verification

### Task 6.1: Update README Credits section

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Find a sensible location (near the end, before License if present).**

Append:

```markdown
## Credits

Portions of the cross-platform skill/hook infrastructure are derived from
[obra/superpowers](https://github.com/obra/superpowers) (MIT, © Jesse Vincent).
See [LICENSE-THIRD-PARTY.md](LICENSE-THIRD-PARTY.md) for the full upstream
license text.

## Install

Reins can be installed as a plugin/extension on several platforms:

- **Claude Code** — via the Claude plugin marketplace (see `.claude-plugin/marketplace.json`)
- **Cursor** — install from this repo's `.cursor-plugin/` directory
- **Gemini CLI** — install via `gemini extension install <repo-url>` (see `gemini-extension.json`)
- **Codex** — manual symlink install, see [.codex/INSTALL.md](.codex/INSTALL.md)
- **OpenCode** — not yet supported (planned)
```

### Task 6.2: Full test run + lint

- [ ] **Step 1: Run full test suite**

```bash
uv run pytest
```

Expected: all green.

- [ ] **Step 2: Run ruff**

```bash
uv run ruff check .
```

Expected: no errors. Fix any that surface.

- [ ] **Step 3: Run pyright**

```bash
uv run pyright
```

Expected: no errors. Fix any that surface.

- [ ] **Step 4: Manual smoke test — invoke bootstrap directly under each env**

```bash
CLAUDE_PLUGIN_ROOT=/fake ./hooks/session-start-bootstrap | python3 -m json.tool | head -5
CURSOR_PLUGIN_ROOT=/fake ./hooks/session-start-bootstrap | python3 -m json.tool | head -5
COPILOT_CLI=1 ./hooks/session-start-bootstrap | python3 -m json.tool | head -5
```

Expected: three different-shaped JSON blobs, each containing escaped `using-reins` content.

### Task 6.3: Commit Phase 6

- [ ] **Step 1: Commit**

```bash
git add README.md
git commit -m "docs: README credits + cross-platform install instructions"
```

---

## Phase 7 — PR

### Task 7.1: Push branch

- [ ] **Step 1:**

```bash
git push -u origin feature/cross-platform-support
```

### Task 7.2: Open PR

- [ ] **Step 1:**

```bash
gh pr create --title "feat: cross-platform support (Cursor, Copilot, Gemini, Codex)" --body "$(cat <<'EOF'
## Summary

Adds first-class support for Cursor, Copilot, Gemini, and Codex alongside Claude Code.

- **Tool-name translations** under `.claude/skills/using-reins/references/` map Claude Code tool names used in skill bodies to each platform's equivalents.
- **Polyglot SessionStart bootstrap** (`hooks/session-start-bootstrap` + `hooks/run-hook.cmd`) detects host platform via env vars and emits `using-reins` content in the right JSON shape per platform.
- **Plugin manifests** (`.claude-plugin/`, `.cursor-plugin/`, `gemini-extension.json`, `.codex/INSTALL.md`) make reins installable from each platform's plugin/extension channel.
- `reins hooks install` registers the SessionStart entry in each platform's config file alongside the existing PreToolUse hooks.
- `reins init` copies the bootstrap hook into end-user projects.

Cross-platform infrastructure patterns adapted from [obra/superpowers](https://github.com/obra/superpowers) (MIT). Full upstream license recorded in `LICENSE-THIRD-PARTY.md`.

**Out of scope (deferred):** OpenCode JS plugin (noted in `kb/reins/exec-plans/active/feature-cross-platform-support/decisions.md`).

## Test plan

- [x] `uv run pytest` — all green
- [x] `uv run ruff check .` — no errors
- [x] `uv run pyright` — no errors
- [x] Manual smoke: `CLAUDE_PLUGIN_ROOT=/fake ./hooks/session-start-bootstrap | python3 -m json.tool` emits Claude shape
- [x] Manual smoke: `CURSOR_PLUGIN_ROOT=/fake ./hooks/session-start-bootstrap | python3 -m json.tool` emits Cursor shape
- [x] Manual smoke: `COPILOT_CLI=1 ./hooks/session-start-bootstrap | python3 -m json.tool` emits SDK-default shape
- [x] `reins init` in a fresh repo with all 4 platforms selected installs SessionStart in every platform config

EOF
)"
```

---

## Self-Review Checklist

**Spec coverage:**
- (1) Tool-name references — Phase 1 ✔
- (2) Polyglot SessionStart hook — Phase 2 ✔
- (3) Plugin manifests — Phase 4 ✔
- Attribution (LICENSE-THIRD-PARTY + Credits + per-file headers) — Phase 0 + Phase 6 ✔
- `reins init` / `reins hooks install` wiring — Phase 3 + Phase 5 ✔
- Out-of-scope (OpenCode) — recorded in decisions.md (Phase 2.1) ✔

**Placeholder scan:** Replace `<VERSION-FROM-PYPROJECT>` in Phase 4 with the actual version from `pyproject.toml`. No other placeholders.

**Type/name consistency:**
- `session-start-bootstrap` (script name) — consistent across Phases 2, 3, 5.
- `_BOOTSTRAP_SCRIPT`, `_CLAUDE_SS_ENTRY`, `_CURSOR_SS_ENTRY`, `_COPILOT_SS_ENTRY` — defined once in Phase 3 Task 3.2 Step 1 and used consistently.
- `event` kwarg on `_merge_*_settings` — defaulted in Phase 3 Task 3.2 Step 2/Step 4 and passed explicitly in Step 3.
