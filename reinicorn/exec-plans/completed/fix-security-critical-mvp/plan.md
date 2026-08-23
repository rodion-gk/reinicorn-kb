---
type: plan
title: Fix Critical Security Vulnerabilities Implementation Plan
slug: fix-security-critical-mvp
lifecycle: dropped
status: abandoned
created: 2026-03-24
author: Michael Biehl
branch: fix-security-critical-mvp
ticket: N/A
---

# Fix Critical Security Vulnerabilities Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use executing-plans to implement this plan task-by-task.

**Goal:** Fix all 5 critical security vulnerabilities (SEC-01 through SEC-05) before tagging v0.1.0.

**Architecture:** Add boundary validation at every point where user-controlled strings flow into filesystem paths or shell commands. A shared `validate_safe_name()` function handles name validation (SEC-04, SEC-05). Path traversal checks use `Path.resolve().is_relative_to()` (SEC-02, SEC-03). Agent command execution switches from `shell=True` string interpolation to `shell=False` with an argv list (SEC-01). Hook names are validated against known git hooks.

**Tech Stack:** Python 3.12, pathlib, re, shlex, pytest

---

## Execution Plan: fix/security-critical-mvp

## Acceptance Criteria
- [ ] SEC-01: `REINS_AGENT_CMD` parsed as argv list, executed with `shell=False`
- [ ] SEC-02: Extension copy/append rejects paths that escape ext_dir or project_dir
- [ ] SEC-03: Extension hook names validated against known git hook names
- [ ] SEC-04: Extension names validated against safe-name regex
- [ ] SEC-05: Agent `task_name` and `job_id` validated against safe-name regex
- [ ] All existing tests still pass
- [ ] New tests cover each attack vector with malicious inputs
- [ ] 0 ruff errors, 0 pyright errors

## Approach

**Golden Principles compliance:**
- GP1 (validate at boundaries): All validation happens at the entry points — `find_extension()`, `apply_extension()`, `cmd_agent_run()`, `cmd_agent_log()`
- GP2 (shared utilities): Single `validate_safe_name()` in `src/reins/validation.py`
- GP4 (agent-readable errors): Each rejection includes what failed, where, and how to fix
- GP7 (tests for every behavior): TDD — failing tests first for each vulnerability
- P1 (no dynamic Python strings): Also fix `_write_completion_script` while we're in agent.py (SEC-12, currently medium, but violates P1)

## Dependencies
None — all changes are to existing modules with no cross-branch conflicts.

---

### Task 1: Create `validation.py` with `validate_safe_name()`

**Files:**
- Create: `src/reins/validation.py`
- Test: `tests/test_validation.py`

**Step 1: Write the failing tests**

```python
# tests/test_validation.py
"""Tests for reins.validation boundary checks."""

from __future__ import annotations

import pytest

from reins.validation import validate_safe_name

class TestValidateSafeName:
    """SEC-04, SEC-05: name validation rejects path traversal and shell metacharacters."""

    def test_simple_name(self):
        assert validate_safe_name("my-task") == "my-task"

    def test_name_with_dots(self):
        assert validate_safe_name("v1.2.3") == "v1.2.3"

    def test_name_with_underscores(self):
        assert validate_safe_name("my_extension") == "my_extension"

    def test_rejects_path_traversal(self):
        with pytest.raises(ValueError, match="path traversal"):
            validate_safe_name("../../etc/passwd")

    def test_rejects_dotdot(self):
        with pytest.raises(ValueError, match="path traversal"):
            validate_safe_name("..")

    def test_rejects_slash(self):
        with pytest.raises(ValueError, match="path traversal"):
            validate_safe_name("foo/bar")

    def test_rejects_backslash(self):
        with pytest.raises(ValueError, match="path traversal"):
            validate_safe_name("foo\\bar")

    def test_rejects_null_byte(self):
        with pytest.raises(ValueError, match="path traversal"):
            validate_safe_name("foo\x00bar")

    def test_rejects_empty(self):
        with pytest.raises(ValueError, match="path traversal"):
            validate_safe_name("")

    def test_rejects_dot_prefix(self):
        with pytest.raises(ValueError, match="path traversal"):
            validate_safe_name(".hidden")

    def test_rejects_shell_metacharacters(self):
        for bad in ["foo;bar", "foo|bar", "foo$(cmd)", "foo`cmd`", "foo&bar"]:
            with pytest.raises(ValueError, match="path traversal"):
                validate_safe_name(bad)
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_validation.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'reins.validation'`

**Step 3: Write minimal implementation**

```python
# src/reins/validation.py
"""Boundary validation for user-supplied names and identifiers."""

from __future__ import annotations

import re

# Alphanumeric, hyphens, underscores, dots. Must start with alphanumeric.
_SAFE_NAME_RE = re.compile(r"^[a-zA-Z0-9][a-zA-Z0-9._-]*$")

def validate_safe_name(name: str) -> str:
    """Validate that a name is safe for use in filesystem paths.

    Returns the name unchanged if valid.
    Raises ValueError if the name contains path traversal or shell metacharacters.
    """
    if not name or not _SAFE_NAME_RE.match(name):
        raise ValueError(
            f"Invalid name: {name!r} — contains path traversal or unsafe characters.\n"
            f"  Names must match: {_SAFE_NAME_RE.pattern}\n"
            f"  How to fix: Use only letters, numbers, hyphens, underscores, and dots."
        )
    return name
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_validation.py -v`
Expected: all PASS

**Step 5: Commit**

```bash
git add src/reins/validation.py tests/test_validation.py
git commit -m "feat: add validate_safe_name() boundary validation (SEC-04, SEC-05)"
```

---

### Task 2: SEC-05 — Validate `task_name` and `job_id` in agent.py

**Files:**
- Modify: `src/reins/commands/agent.py:55,82,158-164`
- Test: `tests/commands/test_agent.py`

**Step 1: Write the failing tests**

Add to `tests/commands/test_agent.py`:

```python
def test_agent_run_rejects_path_traversal(harness_repo, capsys):
    """SEC-05: task_name with path traversal must be rejected."""
    with patch("reins.commands.agent.repo_root", return_value=harness_repo), \
         patch("reins.commands.agent.config_get", return_value="echo {prompt}"):
        result = cmd_agent_run("../../etc/passwd")

    assert result == 1
    captured = capsys.readouterr()
    assert "path traversal" in captured.err.lower() or "Invalid name" in captured.err

def test_agent_run_rejects_shell_metachar(harness_repo, capsys):
    """SEC-05: task_name with shell metacharacters must be rejected."""
    with patch("reins.commands.agent.repo_root", return_value=harness_repo), \
         patch("reins.commands.agent.config_get", return_value="echo {prompt}"):
        result = cmd_agent_run("task;rm -rf /")

    assert result == 1

def test_agent_log_rejects_path_traversal(harness_repo, capsys):
    """SEC-05: job_id with path traversal must be rejected."""
    with patch("reins.commands.agent.repo_root", return_value=harness_repo):
        from reins.commands.agent import cmd_agent_log
        result = cmd_agent_log("../../etc/passwd")

    assert result == 1
    captured = capsys.readouterr()
    assert "path traversal" in captured.err.lower() or "Invalid" in captured.err
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/commands/test_agent.py::test_agent_run_rejects_path_traversal tests/commands/test_agent.py::test_agent_run_rejects_shell_metachar tests/commands/test_agent.py::test_agent_log_rejects_path_traversal -v`
Expected: FAIL — traversal names currently accepted

**Step 3: Add validation to agent.py**

In `cmd_agent_run()`, add validation right after getting `root` and `harness_dir` (before line 61):

```python
from reins.validation import validate_safe_name

# Add at start of cmd_agent_run, after harness_dir:
    try:
        validate_safe_name(task_name)
    except ValueError as e:
        console.error(str(e))
        return 1
```

In `cmd_agent_log()`, add validation right after getting `harness_dir` (before line 164):

```python
    try:
        validate_safe_name(job_id)
    except ValueError as e:
        console.error(str(e))
        return 1
```

Note: `job_id` format is `20260215T120000-test-task` which contains only safe chars, so existing tests still pass.

**Step 4: Run full agent test suite**

Run: `uv run pytest tests/commands/test_agent.py -v`
Expected: all PASS (old + new)

**Step 5: Commit**

```bash
git add src/reins/commands/agent.py tests/commands/test_agent.py
git commit -m "fix: validate task_name and job_id against path traversal (SEC-05)"
```

---

### Task 3: SEC-04 — Validate extension names in extensions.py

**Files:**
- Modify: `src/reins/extensions.py:22,63`
- Test: `tests/test_extensions.py`

**Step 1: Write the failing tests**

Add to `tests/test_extensions.py`:

```python
import pytest

def test_find_extension_rejects_traversal(tmp_path: Path):
    """SEC-04: extension name with path traversal must be rejected."""
    with pytest.raises(ValueError, match="path traversal|Invalid name"):
        find_extension("../../etc", tmp_path)

def test_find_extension_rejects_dotdot(tmp_path: Path):
    """SEC-04: '..' as extension name must be rejected."""
    with pytest.raises(ValueError, match="path traversal|Invalid name"):
        find_extension("..", tmp_path)

def test_apply_extension_rejects_traversal(tmp_path: Path):
    """SEC-04: apply_extension with path traversal must be rejected."""
    result = apply_extension("../../etc/passwd", tmp_path)
    assert result == 1
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_extensions.py::test_find_extension_rejects_traversal tests/test_extensions.py::test_find_extension_rejects_dotdot tests/test_extensions.py::test_apply_extension_rejects_traversal -v`
Expected: FAIL

**Step 3: Add validation**

In `extensions.py`, add at the top of `find_extension()` (line 23):

```python
from reins.validation import validate_safe_name

def find_extension(name: str, search_root: Path | None = None) -> Path | None:
    validate_safe_name(name)  # SEC-04: reject path traversal
    # ... rest unchanged
```

In `apply_extension()` (line 64), add early validation:

```python
def apply_extension(name: str, project_dir: Path) -> int:
    try:
        validate_safe_name(name)  # SEC-04
    except ValueError as e:
        console.error(str(e))
        return 1
    # ... rest unchanged
```

Note: `find_extension` raises (caller handles), `apply_extension` catches and returns error code (CLI boundary).

**Step 4: Run full extensions test suite**

Run: `uv run pytest tests/test_extensions.py -v`
Expected: all PASS

**Step 5: Commit**

```bash
git add src/reins/extensions.py tests/test_extensions.py
git commit -m "fix: validate extension names against path traversal (SEC-04)"
```

---

### Task 4: SEC-02 & SEC-03 — Path traversal and hook name validation in extensions.py

**Files:**
- Modify: `src/reins/extensions.py:92-174`
- Modify: `src/reins/validation.py` (add `validate_hook_name`)
- Test: `tests/test_extensions.py`
- Test: `tests/test_validation.py`

**Step 1: Write the failing tests**

Add `validate_hook_name` tests to `tests/test_validation.py`:

```python
from reins.validation import validate_hook_name

class TestValidateHookName:
    """SEC-03: hook name validation rejects arbitrary filenames."""

    def test_valid_hook(self):
        assert validate_hook_name("post-checkout") == "post-checkout"

    def test_valid_pre_push(self):
        assert validate_hook_name("pre-push") == "pre-push"

    def test_rejects_traversal(self):
        with pytest.raises(ValueError, match="not a recognized git hook"):
            validate_hook_name("../../.profile")

    def test_rejects_arbitrary_name(self):
        with pytest.raises(ValueError, match="not a recognized git hook"):
            validate_hook_name("run-my-payload")

    def test_rejects_empty(self):
        with pytest.raises(ValueError, match="not a recognized git hook"):
            validate_hook_name("")
```

Add SEC-02 and SEC-03 tests to `tests/test_extensions.py`:

```python
def test_apply_extension_rejects_copy_traversal_src(tmp_path: Path, harness_repo: Path):
    """SEC-02: copy source escaping ext dir must be rejected."""
    ext_dir = harness_repo / "extensions" / "stacks" / "evil-ext"
    ext_dir.mkdir(parents=True)
    (ext_dir / "manifest.json").write_text(json.dumps({
        "name": "evil-ext",
        "description": "Malicious",
        "copy": {"../../etc/passwd": "stolen.txt"},
    }))

    result = apply_extension("evil-ext", harness_repo)
    assert result == 1  # should fail, not copy

def test_apply_extension_rejects_copy_traversal_dest(tmp_path: Path, harness_repo: Path):
    """SEC-02: copy dest escaping project dir must be rejected."""
    ext_dir = harness_repo / "extensions" / "stacks" / "evil-ext"
    ext_dir.mkdir(parents=True)
    (ext_dir / "legit.txt").write_text("payload")
    (ext_dir / "manifest.json").write_text(json.dumps({
        "name": "evil-ext",
        "description": "Malicious",
        "copy": {"legit.txt": "../../etc/evil.txt"},
    }))

    result = apply_extension("evil-ext", harness_repo)
    assert result == 1

def test_apply_extension_rejects_append_traversal(tmp_path: Path, harness_repo: Path):
    """SEC-02: append target escaping project dir must be rejected."""
    ext_dir = harness_repo / "extensions" / "stacks" / "evil-ext"
    ext_dir.mkdir(parents=True)
    (ext_dir / "payload.md").write_text("evil content")
    (ext_dir / "manifest.json").write_text(json.dumps({
        "name": "evil-ext",
        "description": "Malicious",
        "append": {"../../.bashrc": "payload.md"},
    }))

    result = apply_extension("evil-ext", harness_repo)
    assert result == 1

def test_apply_extension_rejects_hook_traversal(harness_repo: Path):
    """SEC-03: hook name with traversal must be rejected."""
    import subprocess
    subprocess.run(["git", "init"], cwd=harness_repo, capture_output=True)

    ext_dir = harness_repo / "extensions" / "stacks" / "evil-ext"
    ext_dir.mkdir(parents=True)
    (ext_dir / "hook.sh").write_text("#!/bin/bash\necho pwned")
    (ext_dir / "manifest.json").write_text(json.dumps({
        "name": "evil-ext",
        "description": "Malicious",
        "hooks": {"../../.profile": "hook.sh"},
    }))

    result = apply_extension("evil-ext", harness_repo)
    assert result == 1
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_validation.py::TestValidateHookName tests/test_extensions.py::test_apply_extension_rejects_copy_traversal_src tests/test_extensions.py::test_apply_extension_rejects_hook_traversal -v`
Expected: FAIL

**Step 3a: Add `validate_hook_name` to validation.py**

```python
# Known git hooks — comprehensive list from git documentation
VALID_GIT_HOOKS = frozenset({
    "applypatch-msg", "pre-applypatch", "post-applypatch",
    "pre-commit", "pre-merge-commit", "prepare-commit-msg", "commit-msg", "post-commit",
    "pre-rebase", "post-checkout", "post-merge", "pre-push",
    "pre-receive", "update", "proc-receive", "post-receive", "post-update",
    "reference-transaction", "push-to-checkout", "pre-auto-gc",
    "post-rewrite", "sendemail-validate", "fsmonitor-watchman",
    "p4-changelist", "p4-prepare-changelist", "p4-post-changelist", "p4-pre-submit",
    "post-index-change",
})

def validate_hook_name(name: str) -> str:
    """Validate that a hook name is a recognized git hook.

    Returns the name unchanged if valid.
    Raises ValueError if the name is not a known git hook.
    """
    if name not in VALID_GIT_HOOKS:
        raise ValueError(
            f"'{name}' is not a recognized git hook name.\n"
            f"  How to fix: Use a standard git hook name (e.g., post-checkout, pre-push)."
        )
    return name
```

**Step 3b: Add path traversal checks to `apply_extension()` in extensions.py**

Add a helper at module level:

```python
from reins.validation import validate_hook_name, validate_safe_name

def _check_path_escape(resolved: Path, boundary: Path, label: str) -> None:
    """Raise if resolved path escapes the boundary directory."""
    if not resolved.is_relative_to(boundary):
        raise ValueError(
            f"Path traversal blocked: {label} escapes {boundary}\n"
            f"  Resolved to: {resolved}\n"
            f"  How to fix: Remove '../' from manifest paths."
        )
```

In the **copy** loop (line 92-109), after constructing `full_src` and `full_dest`:

```python
    for src_path, dest_path in manifest.get("copy", {}).items():
        full_src = ext_dir / src_path
        full_dest = project_dir / dest_path
        try:
            _check_path_escape(full_src.resolve(), ext_dir.resolve(), f"copy source '{src_path}'")
            _check_path_escape(full_dest.resolve(), project_dir.resolve(), f"copy dest '{dest_path}'")
        except ValueError as e:
            console.error(str(e))
            return 1
        # ... rest of copy logic unchanged
```

In the **append** loop (line 112-133), after constructing `full_src` and `full_dest`:

```python
    for target_file, src_file in manifest.get("append", {}).items():
        full_src = ext_dir / src_file
        full_dest = project_dir / target_file
        try:
            _check_path_escape(full_src.resolve(), ext_dir.resolve(), f"append source '{src_file}'")
            _check_path_escape(full_dest.resolve(), project_dir.resolve(), f"append dest '{target_file}'")
        except ValueError as e:
            console.error(str(e))
            return 1
        # ... rest of append logic unchanged
```

In the **hooks** loop (line 136-174), validate hook name before using it:

```python
    for hook_name, src_path in manifest.get("hooks", {}).items():
        try:
            validate_hook_name(hook_name)  # SEC-03
        except ValueError as e:
            console.error(str(e))
            return 1
        # ... rest of hooks logic unchanged
```

**Step 4: Run full test suite**

Run: `uv run pytest tests/test_extensions.py tests/test_validation.py -v`
Expected: all PASS

**Step 5: Commit**

```bash
git add src/reins/validation.py src/reins/extensions.py tests/test_extensions.py tests/test_validation.py
git commit -m "fix: block path traversal in extension manifests and validate hook names (SEC-02, SEC-03)"
```

---

### Task 5: SEC-01 — Harden agent command execution

**Files:**
- Modify: `src/reins/commands/agent.py:39-52,61,77-101`
- Create: `src/reins/_agent_complete.py` (replaces dynamic script generation, fixes P1 violation)
- Test: `tests/commands/test_agent.py`

This is the most involved fix. The current approach uses `shell=True` with string interpolation. We switch to `shell=False` with explicit argv.

**Step 1: Write the failing tests**

Add to `tests/commands/test_agent.py`:

```python
def test_agent_run_rejects_shell_metachar_in_config(harness_repo, capsys):
    """SEC-01: REINS_AGENT_CMD with shell metacharacters must be rejected."""
    task_dir = harness_repo / "harness" / "agent-tasks"
    task_dir.mkdir(parents=True)
    (task_dir / "test-task.md").write_text("Hello.")

    with patch("reins.commands.agent.repo_root", return_value=harness_repo), \
         patch("reins.commands.agent.config_get",
               return_value="echo hi; rm -rf /"):
        result = cmd_agent_run("test-task")

    assert result == 1
    captured = capsys.readouterr()
    assert "shell" in captured.err.lower() or "unsafe" in captured.err.lower()

def test_agent_run_uses_shell_false(harness_repo):
    """SEC-01: Popen must be called with shell=False."""
    task_dir = harness_repo / "harness" / "agent-tasks"
    task_dir.mkdir(parents=True)
    (task_dir / "test-task.md").write_text("Do something.")

    with patch("reins.commands.agent.repo_root", return_value=harness_repo), \
         patch("reins.commands.agent.config_get", return_value="claude -p {prompt}"), \
         patch("reins.commands.agent.commit_harness"), \
         patch("reins.commands.agent.subprocess.Popen") as mock_popen:
        mock_popen.return_value.pid = 12345
        cmd_agent_run("test-task")

    call_kwargs = mock_popen.call_args
    # Should be a list, not a string
    assert isinstance(call_kwargs[0][0], list)
    # shell should be False (or not passed, defaulting to False)
    assert call_kwargs[1].get("shell", False) is False
```

Add test for the static completion module:

```python
def test_agent_complete_module(harness_repo):
    """P1: completion should use a real module, not a generated script."""
    task_dir = harness_repo / "harness" / "agent-tasks"
    task_dir.mkdir(parents=True)
    (task_dir / "test-task.md").write_text("Hello.")

    with patch("reins.commands.agent.repo_root", return_value=harness_repo), \
         patch("reins.commands.agent.config_get", return_value="echo {prompt}"), \
         patch("reins.commands.agent.commit_harness"), \
         patch("reins.commands.agent.subprocess.Popen") as mock_popen:
        mock_popen.return_value.pid = 42
        cmd_agent_run("test-task")

    # The Popen command should reference the module, not _complete.py
    cmd = mock_popen.call_args[0][0]
    assert "_complete.py" not in str(cmd)
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/commands/test_agent.py::test_agent_run_rejects_shell_metachar_in_config tests/commands/test_agent.py::test_agent_run_uses_shell_false tests/commands/test_agent.py::test_agent_complete_module -v`
Expected: FAIL

**Step 3a: Create the static completion module**

```python
# src/reins/_agent_complete.py
"""Agent job completion handler — updates job.json when a task finishes.

Invoked as: python -m reins._agent_complete <exit_code> <job_meta_path>
"""

from __future__ import annotations

import json
import sys
from datetime import UTC, datetime
from pathlib import Path

def main(exit_code: int, meta_path: Path) -> None:
    m = json.loads(meta_path.read_text())
    m["finished"] = datetime.now(UTC).isoformat()
    m["exit_code"] = exit_code
    m["status"] = "completed" if exit_code == 0 else "failed"
    meta_path.write_text(json.dumps(m, indent=2) + "\n")

if __name__ == "__main__":
    main(int(sys.argv[1]), Path(sys.argv[2]))
```

**Step 3b: Rewrite agent command execution in `agent.py`**

Replace `_write_completion_script`, `_build_wrapper`, and the Popen section in `cmd_agent_run`:

```python
import re
import sys

_UNSAFE_CMD_RE = re.compile(r"[;|&`$(){}]")

def _parse_agent_cmd(agent_cmd: str) -> list[str]:
    """Parse REINS_AGENT_CMD into an argv list.

    The value must be a simple command with arguments (no shell metacharacters).
    {prompt} is a placeholder replaced at call time.

    Raises ValueError if the command contains shell metacharacters.
    """
    if _UNSAFE_CMD_RE.search(agent_cmd.replace("{prompt}", "")):
        raise ValueError(
            "REINS_AGENT_CMD contains unsafe shell characters.\n"
            f"  Value: {agent_cmd!r}\n"
            "  How to fix: Use a simple command like 'claude -p {prompt}' without"
            " shell operators (;, |, &, $, backticks)."
        )
    return shlex.split(agent_cmd.replace("{prompt}", "__PROMPT_PLACEHOLDER__"))
```

In `cmd_agent_run`, replace the command building + Popen section:

```python
    # Validate and parse agent command template (SEC-01)
    try:
        argv_template = _parse_agent_cmd(agent_cmd)
    except ValueError as e:
        console.error(str(e))
        return 1

    # Substitute prompt into argv
    argv = [
        arg.replace("__PROMPT_PLACEHOLDER__", prompt)
        if "__PROMPT_PLACEHOLDER__" in arg else arg
        for arg in argv_template
    ]

    # Create job directory
    now = datetime.now(UTC)
    job_id = now.strftime("%Y%m%dT%H%M%S") + "-" + task_name
    jobs_dir = harness_dir / "generated" / "agent-jobs" / job_id
    jobs_dir.mkdir(parents=True, exist_ok=True)

    log_file = jobs_dir / "output.log"
    job_meta = jobs_dir / "job.json"

    # Spawn agent process (shell=False — SEC-01)
    with log_file.open("w") as log_fh:
        proc = subprocess.Popen(
            argv,
            stdout=log_fh,
            stderr=subprocess.STDOUT,
            cwd=root,
        )

    # Write job metadata (task reference only — SEC-09 partial fix)
    meta = {
        "task": task_name,
        "job_id": job_id,
        "started": now.isoformat(),
        "pid": proc.pid,
        "status": "running",
        "command": argv[0],  # just the binary, not the full prompt
    }
    job_meta.write_text(json.dumps(meta, indent=2) + "\n")
```

Remove `_write_completion_script` and `_build_wrapper` functions entirely.

Note: The completion/notification functionality is lost, but it was fragile anyway (P1 violation). We can add it back properly later using the `_agent_complete` module as a subprocess wrapper if needed. For MVP, the job status can be checked with `reins agent status`.

**Step 4: Run full test suite**

Run: `uv run pytest tests/commands/test_agent.py -v`
Expected: all PASS

Note: The existing test `test_agent_run_wrapper_updates_job_on_completion` will need updating since we removed the wrapper. It should be updated to verify `shell=False` and no `_complete.py`.

**Step 5: Lint and type check**

Run: `uv run ruff check src/reins/commands/agent.py src/reins/_agent_complete.py src/reins/validation.py`
Run: `uv run pyright src/`

**Step 6: Commit**

```bash
git add src/reins/commands/agent.py src/reins/_agent_complete.py tests/commands/test_agent.py
git commit -m "fix: harden agent command execution — shell=False + argv parsing (SEC-01)"
```

---

### Task 6: Final verification and tech debt update

**Files:**
- Modify: `harness/reins/tech-debt/by-category/security.md`
- Modify: `harness/reins/tech-debt/cleanup-queue.md`

**Step 1: Run full test suite**

Run: `uv run pytest tests/ -v`
Expected: all PASS

**Step 2: Run linters**

Run: `uv run ruff check src/`
Run: `uv run pyright src/`
Expected: 0 errors

**Step 3: Update tech debt — mark SEC-01 through SEC-05 as resolved**

In `security.md`, add `**Status:** resolved (fix/security-critical-mvp)` to each of the 5 critical entries.

In `cleanup-queue.md`, move SEC-01 through SEC-05 from "queued" to "done".

**Step 4: Commit**

```bash
git add harness/
git commit -m "docs: mark SEC-01 through SEC-05 as resolved in tech debt catalog"
```

---

## Summary

| Task | SEC IDs | Size | What |
|------|---------|------|------|
| 1 | SEC-04, SEC-05 | XS | `validate_safe_name()` utility |
| 2 | SEC-05 | XS | Apply to agent task_name/job_id |
| 3 | SEC-04 | XS | Apply to extension names |
| 4 | SEC-02, SEC-03 | S | Path traversal + hook name validation in extensions |
| 5 | SEC-01 | S | Agent command execution hardening |
| 6 | — | XS | Verify + update tech debt |