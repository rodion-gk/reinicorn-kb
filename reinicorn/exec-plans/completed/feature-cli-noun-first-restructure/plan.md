---
type: plan
title: CLI Noun-First Restructure Implementation Plan
slug: feature-cli-noun-first-restructure
lifecycle: done
status: complete
created: 2026-04-25
author: Michael Biehl
branch: feature-cli-noun-first-restructure
ticket: N/A
---

# CLI Noun-First Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the reins CLI to a uniformly noun-first shape, eliminate divergent doc-creation code paths, and consolidate top-level commands into noun groups (`kb`, `mode`, plus per-doc-type groups).

**Architecture:** The change is concentrated in `src/reins/cli.py` (full rewrite of the argparse tree) and `src/reins/commands/doc_create.py` (refactor into per-type `cmd_<type>_create` entry points; remove the `cmd_doc_create` dispatcher). All other command modules keep their internal entry points; only the CLI dispatch layer changes how they're reached. Tests, console messages, README, GETTING-STARTED, AGENTS.md, kb skills, platform-instructions, and editor hooks are updated mechanically to use the new command surface.

**Tech Stack:** Python 3, argparse, pytest, uv (for `uv run`).

**Reference:** Design doc at `kb/reins/design-docs/cli-noun-first-restructure.md`.

---

## Acceptance Criteria

- [ ] `reins --help` lists noun groups (design, spec, debt, idea, plan, retro, principle, kb, mode, init, hooks, update, feedback, help) and no top-level kb verbs (no `reins sync`, `reins publish`, `reins lint`, `reins status`, `reins enable`, `reins disable`, `reins incognito`, `reins idea "..."` shortcut, `reins doc`, `reins kb-git`).
- [ ] `reins plan create` and the corresponding new entry point produce identical files (no divergent template logic).
- [ ] `reins kb sync`, `reins kb publish`, `reins kb status`, `reins kb lint`, `reins kb list`, `reins kb remove-scope`, `reins kb git ...` all work.
- [ ] `reins mode enable`, `reins mode disable`, `reins mode incognito`, `reins mode status` all work.
- [ ] `reins design create "title"`, `reins spec create "title"`, `reins debt create "title"`, `reins idea create "text"`, `reins retro create`, `reins principle add "title"` all work.
- [ ] `reins doc check-path` is removed; the path-check function is reachable only through internal dispatch as `_check-path`.
- [ ] Full pytest suite passes (`uv run pytest`).
- [ ] Every reference in `README.md`, `GETTING-STARTED.md`, `AGENTS.md`, `.claude/skills/`, `platform-instructions/`, `editor-hooks/`, `linters/`, and command source strings (`console.info(...)` etc.) uses the new commands.
- [ ] No remaining grep hits for: `reins sync`, `reins publish`, `reins enable`, `reins disable`, `reins incognito`, `reins doc create`, `reins kb-git`, `reins idea "` (the bare top-level shortcut), `reins lint`, `reins status` outside historical kb docs (`design-docs/`, `exec-plans/completed/`).

---

## Approach

The work proceeds in five phases. Each phase ends with a green test suite.

1. **Consolidate doc creators.** Make the `plan` doc-type creator in `doc_create.py` delegate to `cmd_plan_create` so the divergent code path disappears immediately. Then expose every doc-type creator as a top-level `cmd_<type>_create` function that the new CLI can import directly.
2. **Rewrite the CLI dispatcher.** Replace `cli.py`'s argparse tree with a noun-first structure. Each top-level subcommand registers its own subparser. The `_dispatch` function fans out by `(group, verb)`.
3. **Update internal call sites.** Every `console.info("Run 'reins enable'...")` and equivalent string inside command source updated to the new command surface.
4. **Update tests.** Tests that exercise the old CLI strings or call `cmd_doc_create` directly are migrated to the new entry points. New tests cover new entry points like `cmd_mode_status`.
5. **Update user-facing documentation.** README, GETTING-STARTED, AGENTS.md, kb skills, platform-instructions, editor hooks, and linter messages.

---

## Tasks

### Task 1: Collapse `cmd_doc_create("plan", ...)` to delegate to `cmd_plan_create`

This eliminates the divergent plan-creation code path before any other work. The `_create_plan` function in `doc_create.py` produces a different plan than `cmd_plan_create` does.

**Files:**
- Modify: `src/reins/commands/doc_create.py:54-67` (`_create_plan`), `:135-142` (`_CREATORS` dict), `:225-258` (`cmd_doc_create`)
- Test: `tests/commands/test_doc_create.py` (existing test for plan creation)

- [ ] **Step 1: Read current behavior of `cmd_doc_create("plan", ...)` test**

Open `tests/commands/test_doc_create.py` and locate the plan-creation test if any. Run:
```bash
uv run pytest tests/commands/test_doc_create.py -v -k plan
```
Note which assertions exist. If none, this task only needs to add a delegation test.

- [ ] **Step 2: Write failing test asserting `cmd_doc_create("plan", "x")` calls `cmd_plan_create`**

Add to `tests/commands/test_doc_create.py`:
```python
def test_doc_create_plan_delegates_to_cmd_plan_create():
    from reins.commands.doc_create import cmd_doc_create
    with patch("reins.commands.doc_create.cmd_plan_create") as mock_plan_create:
        mock_plan_create.return_value = 0
        result = cmd_doc_create("plan", "ignored title")
    assert result == 0
    mock_plan_create.assert_called_once()
```

- [ ] **Step 3: Run test, expect failure**

Run:
```bash
uv run pytest tests/commands/test_doc_create.py::test_doc_create_plan_delegates_to_cmd_plan_create -v
```
Expected: FAIL — `cmd_doc_create` currently runs `_create_plan`, not `cmd_plan_create`.

- [ ] **Step 4: Implement delegation in `cmd_doc_create`**

In `src/reins/commands/doc_create.py`, modify `cmd_doc_create` to delegate `plan` (similar to how `idea` already delegates):

```python
def cmd_doc_create(doc_type: str, title: str) -> int:
    if not title.strip():
        console.error("Title is required.")
        return 1

    if doc_type not in REGISTRY:
        console.error(
            f"Unknown doc type '{doc_type}'. "
            f"Valid types: {', '.join(sorted(REGISTRY))}"
        )
        return 1

    # Delegate idea to existing command
    if doc_type == "idea":
        return cmd_idea(title)

    # Delegate plan to canonical implementation
    if doc_type == "plan":
        from reins.commands.plan import cmd_plan_create
        return cmd_plan_create()

    root = repo_root()
    if root is None:
        return 1
    kb_dir = require_kb_dir(root)

    slug = repo_slug()
    repo_dir = kb_dir / slug
    repo_dir.mkdir(parents=True, exist_ok=True)

    author = _get_author()

    creator = _CREATORS[doc_type]
    filepath = creator(repo_dir, title, author)

    console.success(f"Created: {filepath}")
    commit_kb(root, f"doc({doc_type}): {_slugify(title)}")

    return 0
```

Also delete `_create_plan` from `_CREATORS` (it's no longer used):
```python
_CREATORS = {
    "design": _create_design,
    "spec": _create_spec,
    "debt": _create_debt,
    "retro": _create_retro,
    "principle": _create_principle,
}
```

And delete the `_create_plan` function entirely (lines 54–67).

- [ ] **Step 5: Run delegation test, expect pass**

Run:
```bash
uv run pytest tests/commands/test_doc_create.py::test_doc_create_plan_delegates_to_cmd_plan_create -v
```
Expected: PASS.

- [ ] **Step 6: Run full doc_create test file**

Run:
```bash
uv run pytest tests/commands/test_doc_create.py -v
```
Expected: all pass. If any test was previously asserting the old `_create_plan` behavior, update it to mock `cmd_plan_create` instead.

- [ ] **Step 7: Commit**

```bash
git add src/reins/commands/doc_create.py tests/commands/test_doc_create.py
git commit -m "refactor(doc_create): delegate plan creation to cmd_plan_create"
```

---

### Task 2: Add per-type `cmd_<type>_create` entry points

Expose each remaining doc-type creator (`design`, `spec`, `debt`, `retro`, `principle`) as a top-level callable function so the new CLI dispatcher can import them directly. Idea and plan already have their own modules.

**Files:**
- Modify: `src/reins/commands/doc_create.py`
- Test: `tests/commands/test_doc_create.py`

- [ ] **Step 1: Write failing test for `cmd_design_create`**

Add to `tests/commands/test_doc_create.py`:
```python
def test_cmd_design_create_creates_doc(kb_repo: Path):
    from reins.commands.doc_create import cmd_design_create
    with patch("reins.commands.doc_create.repo_root", return_value=kb_repo), \
         patch("reins.commands.doc_create.run_git") as mock_git, \
         patch("reins.commands.doc_create.commit_kb"), \
         patch("reins.commands.doc_create.repo_slug", return_value="testproject"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_design_create("My Feature")
    assert result == 0
    doc = kb_repo / "kb" / "testproject" / "design-docs" / "my-feature.md"
    assert doc.is_file()
```

- [ ] **Step 2: Run test, expect ImportError**

Run:
```bash
uv run pytest tests/commands/test_doc_create.py::test_cmd_design_create_creates_doc -v
```
Expected: FAIL — `cmd_design_create` doesn't exist yet.

- [ ] **Step 3: Add per-type entry points in `doc_create.py`**

At the bottom of `src/reins/commands/doc_create.py`, add:

```python
def _create_typed(doc_type: str, title: str) -> int:
    """Internal helper used by per-type create entry points."""
    if not title.strip():
        console.error("Title is required.")
        return 1

    root = repo_root()
    if root is None:
        return 1
    kb_dir = require_kb_dir(root)

    slug = repo_slug()
    repo_dir = kb_dir / slug
    repo_dir.mkdir(parents=True, exist_ok=True)

    author = _get_author()
    creator = _CREATORS[doc_type]
    filepath = creator(repo_dir, title, author)

    console.success(f"Created: {filepath}")
    commit_kb(root, f"doc({doc_type}): {_slugify(title)}")
    return 0

def cmd_design_create(title: str) -> int:
    return _create_typed("design", title)

def cmd_spec_create(title: str) -> int:
    return _create_typed("spec", title)

def cmd_debt_create(title: str) -> int:
    return _create_typed("debt", title)

def cmd_retro_create() -> int:
    """Create a retro for the current branch (no title — heading derived from branch)."""
    branch = current_branch() or "unknown"
    return _create_typed("retro", f"Retro: {branch}")

def cmd_principle_add(title: str) -> int:
    return _create_typed("principle", title)
```

- [ ] **Step 4: Run test, expect pass**

Run:
```bash
uv run pytest tests/commands/test_doc_create.py::test_cmd_design_create_creates_doc -v
```
Expected: PASS.

- [ ] **Step 5: Add tests for `cmd_spec_create`, `cmd_debt_create`, `cmd_retro_create`, `cmd_principle_add`**

For each, mirror the design test. For `cmd_retro_create`, also patch `current_branch`:
```python
def test_cmd_retro_create_uses_branch(kb_repo: Path):
    from reins.commands.doc_create import cmd_retro_create
    with patch("reins.commands.doc_create.repo_root", return_value=kb_repo), \
         patch("reins.commands.doc_create.run_git") as mock_git, \
         patch("reins.commands.doc_create.commit_kb"), \
         patch("reins.commands.doc_create.current_branch",
               return_value="feature/x"), \
         patch("reins.commands.doc_create.repo_slug",
               return_value="testproject"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_retro_create()
    assert result == 0
    doc = (kb_repo / "kb" / "testproject" / "exec-plans"
           / "completed" / "feature-x" / "retro.md")
    assert doc.is_file()
```

For `cmd_principle_add`:
```python
def test_cmd_principle_add(kb_repo: Path):
    from reins.commands.doc_create import cmd_principle_add
    with patch("reins.commands.doc_create.repo_root", return_value=kb_repo), \
         patch("reins.commands.doc_create.run_git") as mock_git, \
         patch("reins.commands.doc_create.commit_kb"), \
         patch("reins.commands.doc_create.repo_slug", return_value="testproject"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_principle_add("Always test")
    assert result == 0
    doc = kb_repo / "kb" / "testproject" / "golden-principles.md"
    assert doc.is_file()
    assert "Always test" in doc.read_text()
```

- [ ] **Step 6: Run full doc_create test file**

```bash
uv run pytest tests/commands/test_doc_create.py -v
```
Expected: all pass.

- [ ] **Step 7: Commit**

```bash
git add src/reins/commands/doc_create.py tests/commands/test_doc_create.py
git commit -m "refactor(doc_create): expose per-type create entry points"
```

---

### Task 3: Add `cmd_mode_status`

Add the new `mode status` verb that reports the active mode.

**Files:**
- Modify: `src/reins/commands/mode_cmds.py`
- Test: `tests/test_mode.py`

- [ ] **Step 1: Read current `mode.py` to understand `get_mode()` return values**

Read `src/reins/mode.py`. Confirm `get_mode()` returns one of `"enabled" | "disabled" | "incognito"`.

- [ ] **Step 2: Write failing test**

Add to `tests/test_mode.py`:
```python
def test_cmd_mode_status_prints_current_mode(capsys):
    from reins.commands.mode_cmds import cmd_mode_status
    from unittest.mock import patch
    with patch("reins.commands.mode_cmds.get_mode", return_value="enabled"):
        result = cmd_mode_status()
    assert result == 0
    captured = capsys.readouterr().out
    assert "enabled" in captured.lower()
```

- [ ] **Step 3: Run test, expect ImportError**

```bash
uv run pytest tests/test_mode.py::test_cmd_mode_status_prints_current_mode -v
```
Expected: FAIL.

- [ ] **Step 4: Implement `cmd_mode_status`**

In `src/reins/commands/mode_cmds.py`, append:
```python
def cmd_mode_status() -> int:
    current = get_mode()
    console.info(f"Reins mode: {current}")
    return 0
```

- [ ] **Step 5: Run test, expect pass**

```bash
uv run pytest tests/test_mode.py::test_cmd_mode_status_prints_current_mode -v
```
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/reins/commands/mode_cmds.py tests/test_mode.py
git commit -m "feat(mode): add 'mode status' command"
```

---

### Task 4: Rewrite `cli.py` argparse tree to noun-first shape

Replace the existing argparse tree wholesale. Every top-level becomes a noun group; verbs live one level down.

**Files:**
- Modify: `src/reins/cli.py` (full rewrite of `_build_parser` and `_dispatch`)
- Test: smoke-test by running `reins --help` and a sampling of commands

- [ ] **Step 1: Write a smoke test for the new CLI shape**

Create `tests/test_cli_shape.py`:
```python
"""Smoke tests for the noun-first CLI shape."""
from __future__ import annotations

import subprocess
import sys

def _reins(*args, expect_returncode=0):
    result = subprocess.run(
        [sys.executable, "-m", "reins", *args],
        capture_output=True, text=True,
    )
    assert result.returncode == expect_returncode, (
        f"reins {' '.join(args)} failed: {result.stderr}"
    )
    return result

def test_top_level_help_lists_noun_groups():
    r = _reins("--help")
    out = r.stdout
    for noun in ("design", "spec", "debt", "idea", "plan",
                 "retro", "principle", "kb", "mode",
                 "init", "hooks", "update", "feedback"):
        assert noun in out, f"missing noun group: {noun}"

def test_old_top_level_commands_are_removed():
    for old in ("sync", "publish", "lint", "status",
                "enable", "disable", "incognito", "doc", "kb-git"):
        r = _reins(old, expect_returncode=2)
        assert "invalid choice" in r.stderr.lower() or \
               "unrecognized" in r.stderr.lower() or \
               "argument" in r.stderr.lower()

def test_kb_subcommands_listed():
    r = _reins("kb", "--help")
    for verb in ("sync", "publish", "status", "lint",
                 "list", "remove-scope", "git"):
        assert verb in r.stdout, f"missing kb verb: {verb}"

def test_mode_subcommands_listed():
    r = _reins("mode", "--help")
    for verb in ("enable", "disable", "incognito", "status"):
        assert verb in r.stdout, f"missing mode verb: {verb}"

def test_design_subcommands_listed():
    r = _reins("design", "--help")
    assert "create" in r.stdout
```

- [ ] **Step 2: Run smoke tests, expect failures**

```bash
uv run pytest tests/test_cli_shape.py -v
```
Expected: most fail until the rewrite lands.

- [ ] **Step 3: Replace `_build_parser` in `src/reins/cli.py`**

Open `src/reins/cli.py` and replace `_build_parser` with the noun-first version:

```python
def _build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="reins",
        description="reins — agentic engineering knowledgebase CLI",
    )
    parser.add_argument(
        "--version", action="version", version=f"reins {__version__}"
    )
    sub = parser.add_subparsers(dest="command")

    # ── Doc-type groups ────────────────────────────────────
    def _doc_group_with_create_title(name: str, help_text: str):
        g = sub.add_parser(name, help=help_text)
        gs = g.add_subparsers(dest=f"{name}_command")
        gs.required = True
        cp = gs.add_parser("create", help=f"Create a {name} doc")
        cp.add_argument("title", nargs="+", help="Document title")
        return g

    _doc_group_with_create_title("design", "Design doc operations")
    _doc_group_with_create_title("spec", "Product spec operations")
    _doc_group_with_create_title("debt", "Tech debt doc operations")

    # idea: takes free-form text, not a title
    idea_p = sub.add_parser("idea", help="Idea capture")
    idea_sub = idea_p.add_subparsers(dest="idea_command")
    idea_sub.required = True
    idea_create_p = idea_sub.add_parser("create", help="Capture an idea")
    idea_create_p.add_argument("text", nargs="+", help="Idea text")

    # plan: branch-derived; status/complete verbs
    plan_p = sub.add_parser("plan", help="Execution plan operations")
    plan_sub = plan_p.add_subparsers(dest="plan_command")
    plan_sub.required = True
    plan_sub.add_parser("create", help="Create execution plan for current branch")
    plan_sub.add_parser("status", help="Show plan status for current branch")
    plan_complete_p = plan_sub.add_parser("complete", help="Archive plan to completed/")
    plan_complete_p.add_argument(
        "branch", nargs="?", default=None, help="Branch name (default: current)"
    )

    # retro: branch-derived; no title
    retro_p = sub.add_parser("retro", help="Retrospective operations")
    retro_sub = retro_p.add_subparsers(dest="retro_command")
    retro_sub.required = True
    retro_sub.add_parser("create", help="Create retro for current branch")

    # principle: 'add' verb
    principle_p = sub.add_parser("principle", help="Golden principle operations")
    principle_sub = principle_p.add_subparsers(dest="principle_command")
    principle_sub.required = True
    principle_add_p = principle_sub.add_parser("add", help="Append a principle")
    principle_add_p.add_argument("title", nargs="+", help="Principle title")

    # ── Kb group ────────────────────────────────────────────
    kb_p = sub.add_parser("kb", help="Kb operations (sync, publish, status, lint, ...)")
    kb_sub = kb_p.add_subparsers(dest="kb_command")
    kb_sub.required = True
    kb_sub.add_parser("sync", help="Pull latest kb state")
    kb_sub.add_parser("publish", help="Push kb changes (rebase + push)")
    kb_sub.add_parser("status", help="Show kb status and health")
    kb_sub.add_parser("lint", help="Run kb lint rules")
    kb_sub.add_parser("list", help="List repo scopes in the kb")
    kb_remove_p = kb_sub.add_parser("remove-scope", help="Remove a repo scope from the kb")
    kb_remove_p.add_argument("name", help="Scope name to remove")
    kb_remove_p.add_argument("--force", "-f", action="store_true",
                             help="Skip confirmation prompt")
    kb_git_p = kb_sub.add_parser("git", help="Run git commands inside kb directory")
    kb_git_p.add_argument("git_args", nargs=argparse.REMAINDER, help="Git arguments")

    # ── Mode group ──────────────────────────────────────────
    mode_p = sub.add_parser("mode", help="Mode toggles (enable, disable, incognito, status)")
    mode_sub = mode_p.add_subparsers(dest="mode_command")
    mode_sub.required = True
    mode_sub.add_parser("enable", help="Enable hooks and publishing")
    mode_sub.add_parser("disable", help="Disable all hooks and publishing")
    mode_sub.add_parser("incognito", help="Toggle read-only mode")
    mode_sub.add_parser("status", help="Show active mode")

    # ── Top-level (operate on reins itself) ────────────────
    init_p = sub.add_parser("init", help="Set up reins in this repo")
    init_source = init_p.add_mutually_exclusive_group()
    init_source.add_argument("--kb-url", help="Kb repo URL (skip interactive prompt)")
    init_source.add_argument("--local", action="store_true", help="Create local bare repo")
    init_source.add_argument("--create-remote", action="store_true",
                             help="Create a private GitHub repo for the kb")
    init_p.add_argument("--slug", help="Override the auto-derived repo scope name")
    init_p.add_argument("--kb-name", help="Custom name for the GitHub kb repo")

    hooks_p = sub.add_parser("hooks", help="Git hook management")
    hooks_sub = hooks_p.add_subparsers(dest="hooks_command")
    hooks_sub.required = True
    hooks_sub.add_parser("install", help="Install git hooks")

    update_p = sub.add_parser("update", help="Sync assets with installed reins version")
    update_p.add_argument("--diff", dest="diff_target", help="Show diff for a specific asset")

    feedback_p = sub.add_parser("feedback", help="Report a bug or idea")
    feedback_p.add_argument("text", nargs="*", help="Feedback text (will prompt if omitted)")

    sub.add_parser("help", help="Show help")

    return parser
```

- [ ] **Step 4: Replace `_dispatch` in `src/reins/cli.py`**

Replace `_dispatch` with:

```python
def _dispatch(args: argparse.Namespace) -> int:
    cmd = args.command

    # Doc-type groups
    if cmd == "design" and args.design_command == "create":
        from reins.commands.doc_create import cmd_design_create
        return cmd_design_create(" ".join(args.title))

    if cmd == "spec" and args.spec_command == "create":
        from reins.commands.doc_create import cmd_spec_create
        return cmd_spec_create(" ".join(args.title))

    if cmd == "debt" and args.debt_command == "create":
        from reins.commands.doc_create import cmd_debt_create
        return cmd_debt_create(" ".join(args.title))

    if cmd == "idea" and args.idea_command == "create":
        from reins.commands.idea import cmd_idea
        return cmd_idea(" ".join(args.text))

    if cmd == "plan":
        pc = args.plan_command
        if pc == "create":
            from reins.commands.plan import cmd_plan_create
            return cmd_plan_create()
        if pc == "status":
            from reins.commands.plan import cmd_plan_status
            return cmd_plan_status()
        if pc == "complete":
            from reins.commands.plan import cmd_plan_complete
            return cmd_plan_complete(args.branch)

    if cmd == "retro" and args.retro_command == "create":
        from reins.commands.doc_create import cmd_retro_create
        return cmd_retro_create()

    if cmd == "principle" and args.principle_command == "add":
        from reins.commands.doc_create import cmd_principle_add
        return cmd_principle_add(" ".join(args.title))

    # Kb group
    if cmd == "kb":
        kc = args.kb_command
        if kc == "sync":
            from reins.commands.sync import cmd_sync
            return cmd_sync()
        if kc == "publish":
            from reins.commands.publish import cmd_publish
            return cmd_publish()
        if kc == "status":
            from reins.commands.status import cmd_status
            return cmd_status()
        if kc == "lint":
            from reins.commands.lint import cmd_lint
            return cmd_lint()
        if kc == "list":
            from reins.commands.kb_manage import cmd_kb_list
            return cmd_kb_list()
        if kc == "remove-scope":
            from reins.commands.kb_manage import cmd_kb_remove_scope
            return cmd_kb_remove_scope(args.name, force=args.force)
        if kc == "git":
            from reins.commands.kb_git import cmd_kb_git
            return cmd_kb_git(args.git_args)

    # Mode group
    if cmd == "mode":
        mc = args.mode_command
        if mc == "enable":
            from reins.commands.mode_cmds import cmd_enable
            return cmd_enable()
        if mc == "disable":
            from reins.commands.mode_cmds import cmd_disable
            return cmd_disable()
        if mc == "incognito":
            from reins.commands.mode_cmds import cmd_incognito
            return cmd_incognito()
        if mc == "status":
            from reins.commands.mode_cmds import cmd_mode_status
            return cmd_mode_status()

    # Top-level
    if cmd == "init":
        from reins.commands.init import cmd_init
        return cmd_init(
            kb_url=getattr(args, "kb_url", None),
            local=getattr(args, "local", False),
            create_remote=getattr(args, "create_remote", False),
            kb_name=getattr(args, "kb_name", None),
            slug=getattr(args, "slug", None),
        )

    if cmd == "hooks" and args.hooks_command == "install":
        from reins.commands.hooks_install import cmd_hooks_install
        return cmd_hooks_install()

    if cmd == "update":
        from reins.commands.update import cmd_update
        return cmd_update(diff_target=getattr(args, "diff_target", None))

    if cmd == "feedback":
        from reins.commands.feedback import cmd_feedback
        text = " ".join(args.text) if args.text else None
        return cmd_feedback(text)

    return 1
```

- [ ] **Step 5: Update `_dispatch_internal` to add `_check-path`**

Add the `_check-path` branch to `_dispatch_internal` and add `_check-path` to `_INTERNAL_COMMANDS`:

```python
_INTERNAL_COMMANDS = {
    "_hook-check", "_post-checkout", "_pre-push", "_post-merge",
    "_check-path",
}

def _dispatch_internal(argv: list[str]) -> int:
    """Dispatch internal git hook callbacks (not in argparse, hidden from help)."""
    cmd = argv[0]
    rest = argv[1:]

    if cmd == "_hook-check":
        from reins.commands.internal.hook_check import cmd_hook_check
        return cmd_hook_check()

    if cmd == "_post-checkout":
        from reins.commands.internal.post_checkout import cmd_post_checkout
        return cmd_post_checkout(rest)

    if cmd == "_pre-push":
        from reins.commands.internal.pre_push import cmd_pre_push
        return cmd_pre_push()

    if cmd == "_post-merge":
        from reins.commands.internal.post_merge import cmd_post_merge
        return cmd_post_merge()

    if cmd == "_check-path":
        from reins.commands.doc_create import cmd_doc_check_path
        if not rest:
            return 1
        return cmd_doc_check_path(rest[0])

    return 1
```

- [ ] **Step 6: Run smoke tests**

```bash
uv run pytest tests/test_cli_shape.py -v
```
Expected: all pass.

- [ ] **Step 7: Run a sample of new commands manually**

```bash
uv run reins --help
uv run reins kb --help
uv run reins mode --help
uv run reins design --help
```
Verify each shows the expected verbs.

- [ ] **Step 8: Commit**

```bash
git add src/reins/cli.py tests/test_cli_shape.py
git commit -m "feat(cli): rewrite argparse tree to noun-first shape"
```

---

### Task 5: Remove `cmd_doc_create` and `cmd_doc_check_path` exposure

The `cmd_doc_create` dispatcher is no longer reachable from the CLI. It's still used by `cmd_doc_create("plan", ...)` delegation tests — those tests should be deleted since `plan` no longer routes through `cmd_doc_create`. The `cmd_doc_check_path` function remains but is reached via `_check-path` only.

**Files:**
- Modify: `src/reins/commands/doc_create.py` (delete `cmd_doc_create`, keep `cmd_doc_check_path` and the `cmd_<type>_create` functions)
- Modify: `tests/commands/test_doc_create.py` (remove tests for `cmd_doc_create` itself; keep tests for per-type entry points and `cmd_doc_check_path`)

- [ ] **Step 1: List existing usages of `cmd_doc_create`**

Run:
```bash
grep -rn "cmd_doc_create\b" ~/dev/reins/src ~/dev/reins/tests --include="*.py"
```

- [ ] **Step 2: Delete `cmd_doc_create` from `src/reins/commands/doc_create.py`**

Remove the function definition (the body that takes `doc_type` and dispatches). The per-type `cmd_<type>_create` functions (added in Task 2) remain. The `_CREATORS` dict remains (used internally by `_create_typed`).

- [ ] **Step 3: Delete tests that exercise `cmd_doc_create` directly**

In `tests/commands/test_doc_create.py`, remove:
- `test_create_design_doc` (replaced by `test_cmd_design_create_creates_doc`)
- `test_create_spec_doc`, `test_create_debt_doc`, `test_create_retro_doc`, `test_create_principle`, `test_doc_create_plan_delegates_to_cmd_plan_create` (replaced by per-type tests added in Task 2)
- Any other test that imports `cmd_doc_create`

Keep tests for `cmd_doc_check_path` and the per-type `cmd_<type>_create` tests.

- [ ] **Step 4: Run full test file**

```bash
uv run pytest tests/commands/test_doc_create.py -v
```
Expected: all pass; no ImportError on `cmd_doc_create`.

- [ ] **Step 5: Run full suite**

```bash
uv run pytest
```
Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add src/reins/commands/doc_create.py tests/commands/test_doc_create.py
git commit -m "refactor(doc_create): remove cmd_doc_create dispatcher"
```

---

### Task 6: Update internal console messages in command source

Source code has hard-coded strings like `console.info("Run 'reins enable'...")` that reference the old surface. Update each to the new surface.

**Files:**
- Modify: `src/reins/commands/sync.py:35` — `"reins publish"` → `"reins kb publish"`
- Modify: `src/reins/commands/publish.py:22` — `"reins incognito"` → `"reins mode incognito"`
- Modify: `src/reins/commands/publish.py:24` — `"reins enable"` → `"reins mode enable"`
- Modify: `src/reins/commands/publish.py:51-53` — `"reins sync"`, `"reins publish"` → `"reins kb sync"`, `"reins kb publish"`
- Modify: `src/reins/commands/idea.py:16` — `'reins idea "your idea here"'` → `'reins idea create "your idea here"'`
- Modify: `src/reins/commands/init.py:548` — `"reins status"` → `"reins kb status"`
- Modify: `src/reins/commands/mode_cmds.py:18` — `"reins enable"` → `"reins mode enable"`
- Modify: `src/reins/commands/mode_cmds.py:30` — `"reins incognito"` → `"reins mode incognito"`
- Modify: `src/reins/commands/doc_create.py:194-218` — replace all `"reins doc create {doc_type} ..."` strings with `"reins {doc_type} create ..."` (with special-cases for `idea` → `reins idea create` and `principle` → `reins principle add`); replace `"reins idea \"your idea\""` with `"reins idea create \"your idea\""`
- Modify: `src/reins/commands/plan.py:117` — `"reins plan create"` is already correct (unchanged)
- Modify: `src/reins/commands/kb_git.py:1,15` — module docstring and usage string from `reins kb-git` → `reins kb git`

- [ ] **Step 1: Read each file's offending lines**

For each file path above, open and verify the current text matches the listed strings.

- [ ] **Step 2: Update sync.py**

Edit `src/reins/commands/sync.py:35`:
```python
"Resolve conflicts in kb/, then reins kb publish."
```

- [ ] **Step 3: Update publish.py**

Edit `src/reins/commands/publish.py`:
- Line 22: `console.info("Run 'reins mode incognito' to toggle off.")`
- Line 24: `console.info("Run 'reins mode enable' to re-enable.")`
- Lines 51–53 (the multi-line "what to do on push reject" message):
```python
"    1. reins kb sync       # pull and merge remote changes\n"
"    2. ...\n"
"    3. reins kb publish    # retry\n"
```

- [ ] **Step 4: Update idea.py**

Edit `src/reins/commands/idea.py:16`:
```python
console.error('Usage: reins idea create "your idea here"')
```

- [ ] **Step 5: Update init.py**

Edit `src/reins/commands/init.py:548`:
```python
print("Your kb is ready. Run 'reins kb status' to see kb health.")
```

- [ ] **Step 6: Update mode_cmds.py**

Edit `src/reins/commands/mode_cmds.py`:
- Line 18: `console.info("Run 'reins mode enable' to re-enable.")`
- Line 30: `console.info("Run 'reins mode incognito' again to toggle off.")`

- [ ] **Step 7: Update doc_create.py error messages in `cmd_doc_check_path`**

Edit `src/reins/commands/doc_create.py:189-220`. Map old → new:
- `f"reins doc create {doc_type} \"title\""` → `f"reins {doc_type} create \"title\""` for `design`, `spec`, `debt`, `retro`
- `"reins idea \"your idea\""` → `"reins idea create \"your idea\""`
- `"reins doc create idea \"title\""` → `"reins idea create \"title\""`

The block becomes:
```python
    if subdir == REGISTRY["plan"].dir_path:
        filename = parts[-1]
        plan_filename = REGISTRY["plan"].filename.rsplit("/", 1)[-1]
        retro_filename = REGISTRY["retro"].filename.rsplit("/", 1)[-1]
        if filename == plan_filename:
            console.error(
                "Use 'reins plan create' instead of writing kb docs directly."
            )
            return 2
        elif filename == retro_filename:
            console.error(
                "Use 'reins retro create' instead of writing kb docs directly."
            )
            return 2
        return 0

    if subdir in protected:
        doc_type = protected[subdir]
        if doc_type == "idea":
            console.error(
                "Use 'reins idea create \"your idea\"' instead."
            )
        elif doc_type == "principle":
            console.error(
                "Use 'reins principle add \"title\"' instead of writing "
                "kb docs directly."
            )
        else:
            console.error(
                f"Use 'reins {doc_type} create \"title\"' instead of writing "
                f"kb docs directly."
            )
        return 2

    return 0
```

- [ ] **Step 8: Update kb_git.py**

Edit `src/reins/commands/kb_git.py`:
- Line 1: `"""reins kb git — run git commands inside the kb directory."""`
- Line 15: `console.error("Usage: reins kb git <git-args>")`

- [ ] **Step 9: Run full test suite**

```bash
uv run pytest
```
Expected: all pass. Tests that asserted on old console strings will need updating.

- [ ] **Step 10: Commit**

```bash
git add src/reins/commands/
git commit -m "refactor(commands): update console messages to new CLI surface"
```

---

### Task 7: Update `tests/test_guardrail_hook.py` and other test references

Tests that exercise the old command names need to be migrated.

**Files:**
- Modify: `tests/test_guardrail_hook.py:66-67` (uses `reins kb-git`)
- Modify: `tests/commands/test_kb_git.py` (rename file to `test_kb_git.py` content references; update internal command strings)
- Modify: `tests/commands/test_idea.py` (any usage string assertions)
- Modify: `tests/commands/test_sync.py`, `test_publish.py`, `test_status.py` — assertions on console output may reference old commands

- [ ] **Step 1: Grep test code for old command strings**

Run:
```bash
grep -rn "reins sync\|reins publish\|reins enable\|reins disable\|reins incognito\|reins doc create\|reins kb-git\|reins idea \"\|reins lint\|reins status" ~/dev/reins/tests
```

- [ ] **Step 2: Update each match**

For each grep hit, change to the new surface:
- `reins sync` → `reins kb sync`
- `reins publish` → `reins kb publish`
- `reins enable` → `reins mode enable`
- `reins disable` → `reins mode disable`
- `reins incognito` → `reins mode incognito`
- `reins doc create <type>` → `reins <type> create`
- `reins kb-git` → `reins kb git`
- `reins idea "..."` (top-level shortcut) → `reins idea create "..."`
- `reins lint` → `reins kb lint`
- `reins status` → `reins kb status`

In particular, `tests/test_guardrail_hook.py:66-67`:
```python
def test_hook_allows_reins_kb_git():
    """Hook should allow reins kb git commands."""
    r = _run_hook({"command": "uv run reins kb git status"})
```

- [ ] **Step 3: Run full test suite**

```bash
uv run pytest
```
Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add tests/
git commit -m "test: migrate tests to new CLI surface"
```

---

### Task 8: Update `editor-hooks/block-raw-kb-git.sh` and `linters/`

The editor hook and linter messages mention old commands.

**Files:**
- Modify: `editor-hooks/block-raw-kb-git.sh:8,38-40`
- Modify: `linters/rules/kb/provenance.sh:53`

- [ ] **Step 1: Update `block-raw-kb-git.sh`**

Edit `editor-hooks/block-raw-kb-git.sh`:
- Line 8 (comment): `# Allowed: reins kb publish, reins kb sync, reins kb git`
- Line 38: `echo "  reins kb publish      — push kb changes" >&2`
- Line 39: `echo "  reins kb sync         — pull kb changes" >&2`
- Line 40: `echo "  reins kb git …        — escape hatch for raw git" >&2`

If the hook script itself dispatches based on the current commands, also update its detection logic. Read the full file first:
```bash
cat editor-hooks/block-raw-kb-git.sh
```
Look for any pattern matching against `kb-git`, `sync`, `publish`, etc., and update.

- [ ] **Step 2: Update `provenance.sh:53`**

Edit `linters/rules/kb/provenance.sh:53`:
```bash
echo "${rel_path}:1 — Missing provenance field(s): ${joined}. Add frontmatter via 'reins <type> create' or manually add Author/Status/Origin fields."
```

- [ ] **Step 3: Run tests that exercise the hook**

```bash
uv run pytest tests/test_guardrail_hook.py -v
```
Expected: pass.

- [ ] **Step 4: Commit**

```bash
git add editor-hooks/block-raw-kb-git.sh linters/rules/kb/provenance.sh
git commit -m "chore(hooks,linters): update messages to new CLI surface"
```

---

### Task 9: Update user-facing markdown docs

`README.md`, `GETTING-STARTED.md`, `AGENTS.md`, and the `.claude/skills/` files.

**Files:**
- Modify: `README.md` (many references; see grep results from Task 7's prep)
- Modify: `GETTING-STARTED.md:49,51,67-77,94,99`
- Modify: `AGENTS.md:34-50`
- Modify: `.claude/skills/sync-context.md:16,55`
- Modify: `.claude/skills/publish-plan.md:16,34,43,45`
- Modify: `.claude/skills/writing-plans/SKILL.md:20,178`
- Modify: `.claude/skills/using-reins/SKILL.md:25,29`
- Modify: `.claude/skills/brainstorming/SKILL.md:30,112`
- Modify: `.claude/skills/update-superpowers/replacements.yaml:11`

- [ ] **Step 1: Apply mechanical replacements across all files**

For each file in `README.md`, `GETTING-STARTED.md`, `AGENTS.md`, use the Edit tool with `replace_all=true` for each pair below:

| Old | New |
|---|---|
| ``` `reins sync` ``` | ``` `reins kb sync` ``` |
| ``` `reins publish` ``` | ``` `reins kb publish` ``` |
| ``` `reins lint` ``` | ``` `reins kb lint` ``` |
| ``` `reins status` ``` | ``` `reins kb status` ``` |
| ``` `reins enable` ``` | ``` `reins mode enable` ``` |
| ``` `reins disable` ``` | ``` `reins mode disable` ``` |
| ``` `reins incognito` ``` | ``` `reins mode incognito` ``` |
| ``` `reins kb-git` ``` | ``` `reins kb git` ``` |

Also handle bare (un-backticked) occurrences in prose by reading and editing each match individually. `reins doc create <type>` and `reins idea "..."` need contextual replacement (see Step 2).

- [ ] **Step 2: Manual replacements for `reins doc create` and `reins idea`**

Open each file with hits and replace:
- `reins doc create design "<title>"` → `reins design create "<title>"`
- `reins doc create spec "<title>"` → `reins spec create "<title>"`
- `reins doc create debt "<title>"` → `reins debt create "<title>"`
- `reins doc create retro` → `reins retro create`
- `reins doc create plan` → `reins plan create`
- `reins doc create idea` → `reins idea create`
- `reins doc create principle "<title>"` → `reins principle add "<title>"`
- `reins doc create design-doc "<topic>"` → `reins design create "<topic>"`
- `reins doc create exec-plan ...` → `reins plan create` (no title; uses branch)
- `reins idea "..."` → `reins idea create "..."`

Files needing manual passes:
- `README.md` (line 70: `reins idea "Add rate limiting..."`, line 183: table row, line 222, line 406, line 557)
- `GETTING-STARTED.md` (lines 71, 72)
- `AGENTS.md` (lines 36, 48)
- `.claude/skills/using-reins/SKILL.md` (lines 25, 29)
- `.claude/skills/writing-plans/SKILL.md` (line 20)
- `.claude/skills/brainstorming/SKILL.md` (lines 30, 112)
- `platform-instructions/{claude,copilot,gemini,cursor}.md` (line 18 each)
- `.claude/skills/update-superpowers/replacements.yaml` (line 11)

- [ ] **Step 3: Verify with grep**

```bash
grep -rn "reins sync\|reins publish\|reins enable\|reins disable\|reins incognito\|reins doc create\|reins kb-git\|reins lint\|reins status\b" ~/dev/reins --include="*.md" --include="*.yaml" 2>/dev/null \
  | grep -v ".git/" \
  | grep -v "kb/reins/exec-plans/completed" \
  | grep -v "kb/reins/design-docs"
```

Expected: no hits (the historical kb docs are intentionally untouched).

Also check:
```bash
grep -rn "reins idea \"" ~/dev/reins --include="*.md" 2>/dev/null \
  | grep -v ".git/" \
  | grep -v "kb/reins/exec-plans/completed" \
  | grep -v "kb/reins/design-docs"
```
Expected: no hits (all should now read `reins idea create "..."`).

- [ ] **Step 4: Commit**

```bash
git add README.md GETTING-STARTED.md AGENTS.md .claude/ platform-instructions/
git commit -m "docs: update all references to new CLI surface"
```

---

### Task 10: Update the in-repo `using-reins` skill commands table (if present)

The `using-reins` skill is the primary in-skill reference for the CLI. Its quick-reference must reflect the new shape verbatim.

**Files:**
- Modify: `.claude/skills/using-reins/SKILL.md`

- [ ] **Step 1: Read the current skill content**

Read the full file. Identify the command table or quick-reference list.

- [ ] **Step 2: Rewrite the table to match the new surface**

Replace the quick-reference with the new commands. Recommended structure:

```markdown
| Command | Purpose |
|---|---|
| `reins kb sync` | Pull latest kb state |
| `reins kb publish` | Push kb changes |
| `reins kb status` | Kb health report |
| `reins kb lint` | Run kb lint rules |
| `reins kb git ...` | Raw git passthrough inside the kb |
| `reins design create "title"` | Create a design doc |
| `reins spec create "title"` | Create a product spec |
| `reins debt create "title"` | Create a tech-debt doc |
| `reins idea create "text"` | Capture an idea |
| `reins plan create` | Create execution plan for current branch |
| `reins plan status` | Plan status for current branch |
| `reins plan complete [branch]` | Archive plan to completed/ |
| `reins retro create` | Create retro for current branch |
| `reins principle add "title"` | Append a golden principle |
| `reins mode enable` / `disable` / `incognito` / `status` | Mode toggles |
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/using-reins/SKILL.md
git commit -m "docs(using-reins): rewrite command reference for new CLI surface"
```

---

### Task 11: Final verification

End-to-end check that nothing was missed.

- [ ] **Step 1: Run the entire test suite**

```bash
uv run pytest
```
Expected: all green.

- [ ] **Step 2: Smoke-run the CLI**

```bash
uv run reins --help
uv run reins kb --help
uv run reins mode --help
uv run reins design --help
uv run reins plan --help
uv run reins idea --help
uv run reins principle --help
```
Each shows the expected verbs and no errors.

- [ ] **Step 3: Verify old commands fail cleanly**

```bash
uv run reins sync 2>&1 | head -5     # should error: "invalid choice"
uv run reins doc 2>&1 | head -5      # should error: "invalid choice"
uv run reins kb-git 2>&1 | head -5   # should error: "invalid choice"
```

- [ ] **Step 4: Final grep sweep**

```bash
grep -rn "reins sync\|reins publish\|reins enable\|reins disable\|reins incognito\|reins doc create\|reins kb-git\|reins lint\|reins status\b" \
  ~/dev/reins \
  --include="*.md" --include="*.py" --include="*.sh" --include="*.yaml" \
  2>/dev/null \
  | grep -v ".git/" \
  | grep -v "kb/reins/exec-plans/completed" \
  | grep -v "kb/reins/design-docs"
```
Expected: no hits.

```bash
grep -rn "reins idea \"" \
  ~/dev/reins \
  --include="*.md" --include="*.py" --include="*.sh" \
  2>/dev/null \
  | grep -v ".git/" \
  | grep -v "kb/reins/exec-plans/completed" \
  | grep -v "kb/reins/design-docs"
```
Expected: no hits.

- [ ] **Step 5: Mark plan complete**

```bash
uv run reins plan complete
```

- [ ] **Step 6: Publish kb**

```bash
uv run reins kb publish
```

---

## Dependencies

This plan depends on no other in-flight branches. The `feature/cross-platform-support` branch is empty (no divergence from main) and can remain in place or be deleted.

The kb pointer for the design doc commit (`44fe87e doc(design): cli-noun-first-restructure — full spec`) is already on main and brought along by this branch's first commit (`MM kb` in the parent's `git status`).
