---
type: plan
title: MVP Unified Attach — Implementation Plan
slug: 2026-02-19-mvp-unified-attach-plan
lifecycle: done
status: done
created: 2026-07-17
author: Michael Biehl
branch: mvp-unified-attach
---

# MVP Unified Attach — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Ship an MVP where teammates can `pip install` reins, run `reins attach` in their repo, and get working agent instructions + git hooks.

**Architecture:** Python CLI rewrite of bash attach workflow. Submodule-only harness setup. Multi-tool agent instruction installation via `.claude/skills/` (universal format). New `reins feedback` command for bug reporting.

**Tech Stack:** Python 3.13+, hatchling build, pytest, ruff, pyright

---

### Task 1: Add PyPI metadata to pyproject.toml

**Files:**
- Modify: `pyproject.toml`

**Step 1: Write the failing test**

No automated test needed — this is metadata-only. Verify manually.

**Step 2: Add metadata**

In `pyproject.toml`, add these fields to `[project]`:

```toml
[project]
name = "reins"
version = "0.1.0"
description = "Agentic engineering harness CLI"
requires-python = ">=3.13"
dependencies = []
readme = "README.md"
license = {text = "MIT"}
authors = [{name = "Michael Biehl"}]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.13",
    "Topic :: Software Development :: Quality Assurance",
]

[project.urls]
Repository = "https://github.com/mbiehl/reins"
```

**Step 3: Verify the package builds**

Run: `uv build`
Expected: wheel and sdist created in `dist/`

**Step 4: Verify install from local**

Run: `uv pip install -e .`
Run: `reins --version`
Expected: `reins 0.1.0`

**Step 5: Commit**

```bash
git add pyproject.toml
git commit -m "chore(packaging): add PyPI metadata for pip-installable distribution"
```

---

### Task 2: Remove inline mode from attach command

The design specifies submodule-only. Remove inline mode and extension/variant selection from `cmd_attach()`.

**Files:**
- Modify: `src/reins/commands/attach.py`
- Test: `tests/commands/test_attach.py` (create)

**Step 1: Write failing tests for the trimmed attach flow**

Create `tests/commands/test_attach.py`:

```python
"""Tests for reins attach command."""

from __future__ import annotations

import subprocess
from pathlib import Path
from unittest.mock import patch, call

from reins.commands.attach import cmd_attach

def test_attach_rejects_non_git_dir(tmp_path: Path, capsys):
    """attach should fail if not in a git repo."""
    with patch("reins.commands.attach.run_git") as mock_git:
        mock_git.side_effect = subprocess.CalledProcessError(128, "git")
        with patch("reins.commands.attach.reins_root", return_value=tmp_path):
            result = cmd_attach()
    assert result == 1

def test_attach_does_not_offer_inline_mode(harness_repo: Path, monkeypatch, capsys):
    """attach should not prompt for inline vs submodule — submodule only."""
    # Capture all input() calls to verify no inline option offered
    inputs = []

    def fake_input(prompt):
        inputs.append(prompt)
        if "repo URL" in prompt.lower() or "url" in prompt.lower():
            return "https://github.com/test/harness.git"
        if "1/2" in prompt:
            # This prompt should NOT appear anymore
            raise AssertionError("attach should not offer inline vs submodule choice")
        return ""

    monkeypatch.setattr("builtins.input", fake_input)

    with patch("reins.commands.attach.run_git") as mock_git, \
         patch("reins.commands.attach.reins_root", return_value=harness_repo):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout=str(harness_repo) + "\n"
        )
        # Will fail until we remove inline mode — that's the point
        try:
            cmd_attach()
        except (SystemExit, Exception):
            pass

    # Verify no "1/2" choice was prompted
    for inp in inputs:
        assert "inline" not in inp.lower()
```

**Step 2: Run test to verify it fails**

Run: `uv run pytest tests/commands/test_attach.py -v`
Expected: FAIL (the "1/2" prompt still exists)

**Step 3: Remove inline mode from attach.py**

In `src/reins/commands/attach.py`:
- Remove the `_setup_inline()` function entirely
- Remove the harness mode prompt ("1) Git submodule / 2) Inline")
- Call `_setup_submodule()` directly (it's the only mode now)
- Remove the extension/variant selection block (lines 77-102)
- Remove the `from reins.extensions import ...` import
- Keep the rest of the flow intact

The function should flow as:
1. Verify git repo
2. `_setup_submodule()`
3. Copy AGENTS.md (drop CLAUDE.md — that's Claude Code specific)
4. Copy `.claude/skills/` directory
5. Copy supporting dirs (linters, hooks)
6. Install git hooks (Python, not bash — see Task 4)
7. Print next steps

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/commands/test_attach.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add src/reins/commands/attach.py tests/commands/test_attach.py
git commit -m "refactor(attach): remove inline mode and extension selection

Submodule is the only harness mode. Extensions deferred from MVP."
```

---

### Task 3: Add .claude/skills/ copying to attach

During attach, copy the `.claude/skills/` directory to the target repo so Cursor, Copilot, and Claude Code all discover the skills.

**Files:**
- Modify: `src/reins/commands/attach.py`
- Modify: `tests/commands/test_attach.py`

**Step 1: Write the failing test**

Add to `tests/commands/test_attach.py`:

```python
def test_attach_copies_skills_directory(harness_repo: Path, tmp_path, monkeypatch):
    """attach should copy .claude/skills/ to the target repo."""
    # Set up a fake reins root with skills
    r_root = tmp_path / "reins"
    r_root.mkdir()
    skills_src = r_root / ".claude" / "skills"
    skills_src.mkdir(parents=True)
    (skills_src / "create-exec-plan.md").write_text("# Create Exec Plan\n")
    (skills_src / "sync-context.md").write_text("# Sync Context\n")

    # Target repo should get the skills after attach
    target = harness_repo

    with patch("reins.commands.attach.reins_root", return_value=r_root), \
         patch("reins.commands.attach._setup_submodule"), \
         patch("reins.commands.attach._install_hooks"), \
         patch("reins.commands.attach.run_git") as mock_git:
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout=str(target) + "\n"
        )
        cmd_attach()

    dest_skills = target / ".claude" / "skills"
    assert dest_skills.is_dir()
    assert (dest_skills / "create-exec-plan.md").is_file()
    assert (dest_skills / "sync-context.md").is_file()
```

**Step 2: Run test to verify it fails**

Run: `uv run pytest tests/commands/test_attach.py::test_attach_copies_skills_directory -v`
Expected: FAIL (skills aren't copied yet)

**Step 3: Add skills copying to cmd_attach()**

In `cmd_attach()`, after copying AGENTS.md, add a block that copies `.claude/skills/` from the reins root to the target directory. Use `shutil.copytree` with `dirs_exist_ok=True` to handle re-runs.

**Step 4: Run tests**

Run: `uv run pytest tests/commands/test_attach.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add src/reins/commands/attach.py tests/commands/test_attach.py
git commit -m "feat(attach): copy .claude/skills/ for multi-tool agent support

Skills are read by Cursor, Copilot, and Claude Code from this location."
```

---

### Task 4: Switch attach to Python hook installation

The current `cmd_attach()` calls `scripts/setup-hooks.sh` via subprocess. Switch to calling the existing Python `cmd_hooks_install()` directly.

**Files:**
- Modify: `src/reins/commands/attach.py`
- Modify: `tests/commands/test_attach.py`

**Step 1: Write the failing test**

Add to `tests/commands/test_attach.py`:

```python
def test_attach_uses_python_hooks_install(harness_repo: Path, tmp_path, monkeypatch):
    """attach should call Python hooks_install, not bash setup-hooks.sh."""
    r_root = tmp_path / "reins"
    r_root.mkdir()
    (r_root / ".claude" / "skills").mkdir(parents=True)
    (r_root / "AGENTS.md").write_text("# AGENTS\n")

    with patch("reins.commands.attach.reins_root", return_value=r_root), \
         patch("reins.commands.attach._setup_submodule"), \
         patch("reins.commands.attach.run_git") as mock_git, \
         patch("reins.commands.attach.cmd_hooks_install") as mock_hooks:
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout=str(harness_repo) + "\n"
        )
        mock_hooks.return_value = 0
        cmd_attach()

    mock_hooks.assert_called_once()
```

**Step 2: Run test to verify it fails**

Run: `uv run pytest tests/commands/test_attach.py::test_attach_uses_python_hooks_install -v`
Expected: FAIL

**Step 3: Replace bash subprocess call with Python import**

In `cmd_attach()`:
- Remove the `subprocess.run(["bash", ...setup-hooks.sh...])` block
- Replace with: `from reins.commands.hooks_install import cmd_hooks_install`
- Call `cmd_hooks_install()` directly
- Extract hook installation to a helper `_install_hooks()` for mockability

**Step 4: Run tests**

Run: `uv run pytest tests/commands/test_attach.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add src/reins/commands/attach.py tests/commands/test_attach.py
git commit -m "refactor(attach): use Python hooks_install instead of bash script"
```

---

### Task 5: Clean up git hook scripts — remove bin/reins fallback

The hook shell scripts (`hooks/post-checkout`, `hooks/pre-push`, `hooks/post-merge`) have dead fallback code that checks for `bin/reins`. Remove it.

**Files:**
- Modify: `hooks/post-checkout`
- Modify: `hooks/pre-push`
- Modify: `hooks/post-merge`

**Step 1: No test needed — these are shell scripts with trivial changes**

**Step 2: Clean up post-checkout (lines 29-38)**

Replace the three-way check:

```bash
if command -v reins &>/dev/null; then
    reins _hook-check 2>/dev/null || exit 0
elif [[ -x "bin/reins" ]]; then
    bin/reins _hook-check 2>/dev/null || exit 0
else
    # Check mode file directly
    if [[ -f ".reins-mode" ]] && [[ "$(cat .reins-mode 2>/dev/null)" == "disabled" ]]; then
        exit 0
    fi
fi
```

With:

```bash
if ! command -v reins &>/dev/null; then
    exit 0
fi
reins _hook-check 2>/dev/null || exit 0
```

**Step 3: Clean up pre-push (lines 21-25)**

Replace:

```bash
if command -v reins &>/dev/null; then
    reins _pre-push 2>/dev/null || true
elif [[ -x "bin/reins" ]]; then
    bin/reins _pre-push 2>/dev/null || true
fi
```

With:

```bash
if command -v reins &>/dev/null; then
    reins _pre-push 2>/dev/null || true
fi
```

**Step 4: Clean up post-merge (lines 14-19)**

Replace:

```bash
if command -v reins &>/dev/null; then
    reins _post-merge 2>/dev/null || true
elif [[ -x "bin/reins" ]]; then
    bin/reins _post-merge 2>/dev/null || true
fi
```

With:

```bash
if command -v reins &>/dev/null; then
    reins _post-merge 2>/dev/null || true
fi
```

**Step 5: Commit**

```bash
git add hooks/post-checkout hooks/pre-push hooks/post-merge
git commit -m "fix(hooks): remove dead bin/reins fallback paths

bin/reins was removed during the Python CLI rewrite. Hooks now
require reins to be on PATH (installed via pip)."
```

---

### Task 6: Add `reins feedback` command

New command for teammates to report bugs/ideas on the reins repo.

**Files:**
- Create: `src/reins/commands/feedback.py`
- Create: `tests/commands/test_feedback.py`
- Modify: `src/reins/cli.py` (register command)

**Step 1: Write failing tests**

Create `tests/commands/test_feedback.py`:

```python
"""Tests for reins feedback command."""

from __future__ import annotations

import subprocess
from unittest.mock import patch, MagicMock

from reins.commands.feedback import cmd_feedback, _build_issue_body

def test_build_issue_body_includes_version():
    body = _build_issue_body("Something is broken")
    assert "reins" in body.lower()
    assert "0.1.0" in body or "version" in body.lower()

def test_feedback_inline_text():
    """feedback with inline text should not prompt for description."""
    with patch("reins.commands.feedback._open_issue") as mock_open:
        mock_open.return_value = 0
        result = cmd_feedback("hooks aren't firing")
    assert result == 0
    mock_open.assert_called_once()
    title = mock_open.call_args[0][0]
    assert "hooks" in title.lower()

def test_feedback_no_text_prompts(monkeypatch):
    """feedback with no text should prompt interactively."""
    monkeypatch.setattr("builtins.input", lambda prompt: "my bug report title")
    with patch("reins.commands.feedback._open_issue") as mock_open:
        mock_open.return_value = 0
        result = cmd_feedback(None)
    assert result == 0
    mock_open.assert_called_once()
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/commands/test_feedback.py -v`
Expected: FAIL (module doesn't exist)

**Step 3: Implement feedback.py**

Create `src/reins/commands/feedback.py`:

```python
"""reins feedback — open a GitHub issue on the reins repo."""

from __future__ import annotations

import os
import platform
import shutil
import subprocess
import sys
import urllib.parse
import webbrowser

from reins import __version__, console
from reins.config import get_config_value
from reins.mode import get_mode

REPO_URL = "https://github.com/mbiehl/reins"

def cmd_feedback(text: str | None = None) -> int:
    if text is None:
        print()
        console.header("Reins Feedback")
        print()
        text = input("Describe the issue or idea: ").strip()
        if not text:
            console.error("Feedback text cannot be empty.")
            return 1

    title = text[:80]
    body = _build_issue_body(text)
    return _open_issue(title, body)

def _build_issue_body(description: str) -> str:
    mode = get_mode()
    py_version = f"{sys.version_info.major}.{sys.version_info.minor}.{sys.version_info.micro}"
    lines = [
        "## Description",
        "",
        description,
        "",
        "## Environment",
        "",
        f"- reins: {__version__}",
        f"- Python: {py_version}",
        f"- OS: {platform.system()} {platform.release()}",
        f"- Mode: {mode}",
    ]
    return "\n".join(lines)

def _open_issue(title: str, body: str) -> int:
    if shutil.which("gh"):
        r = subprocess.run(
            ["gh", "issue", "create",
             "--repo", REPO_URL,
             "--title", title,
             "--body", body,
             "--label", "user-feedback"],
            check=False,
        )
        if r.returncode == 0:
            console.success("Issue created.")
            return 0
        console.warn("gh failed — falling back to browser.")

    params = urllib.parse.urlencode({"title": title, "body": body, "labels": "user-feedback"})
    url = f"{REPO_URL}/issues/new?{params}"
    console.info(f"Opening: {url}")
    webbrowser.open(url)
    console.success("Opened issue form in browser.")
    return 0
```

**Step 4: Register in cli.py**

In `src/reins/cli.py`, in `_build_parser()`, add after the `help` subparser:

```python
feedback_p = sub.add_parser("feedback", help="Report a bug or idea")
feedback_p.add_argument("text", nargs="*", help="Feedback text (optional, will prompt if omitted)")
```

In `_dispatch()`, add:

```python
if cmd == "feedback":
    from reins.commands.feedback import cmd_feedback
    text = " ".join(args.text) if args.text else None
    return cmd_feedback(text)
```

**Step 5: Run tests**

Run: `uv run pytest tests/commands/test_feedback.py -v`
Expected: PASS

**Step 6: Commit**

```bash
git add src/reins/commands/feedback.py tests/commands/test_feedback.py src/reins/cli.py
git commit -m "feat(feedback): add reins feedback command for issue reporting

Opens a GitHub issue pre-filled with system context. Uses gh CLI if
available, falls back to browser URL."
```

---

### Task 7: Update AGENTS.md for the new commands

The root AGENTS.md should document `reins feedback` and the updated `reins attach` behavior (submodule-only, copies skills).

**Files:**
- Modify: `AGENTS.md`

**Step 1: Update the CLI table**

Add `reins feedback "text"` to the command table with purpose "Report a bug or idea".

**Step 2: Update the Hard Rules section if needed**

No changes needed — existing rules still apply.

**Step 3: Commit**

```bash
git add AGENTS.md
git commit -m "docs(agents): document feedback command and updated attach behavior"
```

---

### Task 8: Run full test suite and fix any issues

**Step 1: Run full test suite**

Run: `uv run pytest -v`
Expected: All tests pass

**Step 2: Run linters**

Run: `uv run ruff check src/ tests/`
Run: `uv run pyright`
Expected: Clean output

**Step 3: Fix any failures**

Address any test failures or lint issues from the changes above.

**Step 4: Commit fixes if any**

```bash
git add -A
git commit -m "fix: address test and lint issues from MVP changes"
```

---

### Task 9: Manual integration test

This is not automated — it validates the end-to-end flow.

**Step 1: Install reins from the repo**

```bash
cd /tmp
git clone <reins-repo-url> reins-test
cd reins-test
uv pip install -e .
reins --version
```

Expected: `reins 0.1.0`

**Step 2: Create a test repo and attach**

```bash
mkdir /tmp/test-project && cd /tmp/test-project
git init
reins attach
```

Expected: Interactive flow runs, prompts for harness repo, sets up submodule, copies skills, installs hooks.

**Step 3: Verify skills are present**

```bash
ls .claude/skills/
```

Expected: 7 skill files present

**Step 4: Verify hooks are installed**

```bash
ls .git/hooks/post-checkout .git/hooks/pre-push .git/hooks/post-merge
```

Expected: All three hooks present

**Step 5: Test feedback command**

```bash
reins feedback "test issue from integration test"
```

Expected: Opens GitHub issue or browser

**Step 6: Clean up**

```bash
rm -rf /tmp/test-project /tmp/reins-test
```
