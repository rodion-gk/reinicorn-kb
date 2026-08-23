---
type: plan
title: 'Execution Plan: feature-axi-output-surface'
slug: feature-axi-output-surface
lifecycle: done
status: complete
created: 2026-07-02
author: Michael Biehl
branch: feature-axi-output-surface
ticket: N/A
---

# Execution Plan: feature-axi-output-surface

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

## Goal

Implement the agent-native output surface spec
(`kb/reins/specs/agent-native-output-surface-axi-principles.md`): axi channel
model (errors on stdout, stderr = progress/debug only), `next:` footer
convention, per-type `show`/`list` verbs with truncation + `--full`,
content-first bare `reins` home view, `kb status --compact` wired into
session-start, and idempotent no-op mutations.

## Acceptance Criteria

- [x] `console.error` writes `error: <msg>` to **stdout**; new `console.progress` writes to stderr; new `console.next_step` prints `next: <command>` lines to stdout
- [x] Indentation/bold decoration only when stdout is a TTY (content identical either way)
- [x] `reins spec|prd|debt|idea show <slug> [--full]` and `... list` work; preview truncates at 1500 chars with `… (truncated, N chars total)` + `next:` hint; unknown slug exits 1 with valid slugs inline
- [x] `reins plan show [branch] [--full]` and `reins retro show [branch] [--full]` work
- [x] Bare `reins` prints bin path, description, live branch/plan/overlap state, and conditional `next:` suggestions — local reads only, never the argparse manual; `reins help` / `--help` keep the manual
- [x] `reins kb status --compact` prints a ≤10-line undecorated dashboard; `session-start.sh` emits it
- [x] `mode enable|disable` re-runs say `(no-op)` and exit 0; `hooks install` re-run summarizes as no-op; `update` re-run verified exit 0
- [x] `tests/test_output_conventions.py` structurally enforces the channel + shape rules
- [x] Full test suite (`uv run pytest`), `uv run ruff check`, `uv run pyright` all pass

## Approach

Bottom-up: console channel primitives first (everything depends on them), then
the shared `doc_show` module + CLI wiring, then the home view, then compact
status + hook, then footers/idempotency polish on existing commands, then
structural enforcement tests. TDD per task; conventional commits per task.
All work on branch `feature-axi-output-surface`; kb docs (this plan) publish
via `reins kb publish`.

Key codebase facts for workers with zero context:

- Entry point: `src/reins/cli.py` — argparse builder `_build_parser()`, dict
  `_DISPATCH` mapping `(noun, verb)` → lazy-loaded handler, `main()`.
- Doc metadata: `src/reins/doc_types.py` `REGISTRY` (dir_path, filename
  patterns like `"active/{branch}/plan.md"`).
- Output helpers: `src/reins/console.py` (info/success/warn/error/header).
- Command modules: `src/reins/commands/*.py`; handlers return int exit codes.
- Tests: pytest, fixtures in `tests/conftest.py` (`kb_repo` gives a temp git
  repo with a `kb/testproject/` scope), run with `uv run pytest`.
- Git helpers: `src/reins/git.py` (`repo_root(quiet=False)`, `current_branch()`,
  `repo_slug()`, `sanitize_branch()`); kb helpers: `src/reins/kb.py`
  (`get_kb_dir(root)`, `require_kb_dir(root)`, `check_overlap(branch, root)`).

---

## Tasks

### Task 1: console channel model (`error` → stdout, `progress`, `next_step`)

**Files:**
- Modify: `src/reins/console.py`
- Test: `tests/test_console.py`

- [x] **Step 1: Write failing tests**

Append to `tests/test_console.py`:

```python
def test_error_is_structured_and_on_stdout(capsys):
    console.error("kb not found")
    out, err = capsys.readouterr()
    assert "error: kb not found" in out
    assert err == ""

def test_progress_goes_to_stderr(capsys):
    console.progress("Publishing kb changes...")
    out, err = capsys.readouterr()
    assert out == ""
    assert "Publishing kb changes..." in err

def test_next_step_prints_one_line_per_command(capsys):
    console.next_step("reins plan create", "reins kb status")
    out, err = capsys.readouterr()
    assert out == "next: reins plan create\nnext: reins kb status\n"
    assert err == ""
```

(Match the existing import style at the top of `tests/test_console.py` —
it already imports `console`.)

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_console.py -v`
Expected: FAIL — `AttributeError: module 'reins.console' has no attribute 'progress'` (and the error-channel assert fails).

- [x] **Step 3: Implement in `src/reins/console.py`**

Replace `error()` and add the two new functions:

```python
def error(msg: str) -> None:
    """Structured error on stdout (agents read stdout; exit codes carry status)."""
    print(_c(_RED, f"error: {msg}"))

def progress(msg: str) -> None:
    """Progress/debug diagnostic on stderr — never data."""
    print(msg, file=sys.stderr)

def next_step(*commands: str) -> None:
    """Axi contextual disclosure: one `next: <command>` line per suggestion."""
    for cmd in commands:
        print(f"next: {cmd}")
```

Update the module docstring to state the channel model: stdout = data,
errors, suggestions; stderr = progress/debug; exit codes = status.

- [x] **Step 4: Run the full suite and fix stderr-based assertions**

Run: `uv run pytest`
Expected: new console tests PASS; any existing test asserting error text in
`capsys.readouterr().err` now fails — update those assertions to read `.out`
and to expect the `error: ` prefix. (`grep -rn "\.err" tests/` to find them.)

- [x] **Step 5: Commit**

```bash
git add src/reins/console.py tests/
git commit -m "feat(console): axi channel model — errors to stdout, progress channel, next_step helper"
```

### Task 2: decoration gated on TTY

**Files:**
- Modify: `src/reins/console.py`
- Test: `tests/test_console.py`

- [x] **Step 1: Write failing test**

```python
def test_indentation_is_tty_only(capsys):
    # pytest capture is not a TTY, so output must be undecorated
    console.info("Kb: /tmp/kb")
    out, _ = capsys.readouterr()
    assert out == "Kb: /tmp/kb\n"
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_console.py::test_indentation_is_tty_only -v`
Expected: FAIL — output is `"  Kb: /tmp/kb\n"` (two-space indent).

- [x] **Step 3: Implement**

In `src/reins/console.py`, add a TTY check (separate from `_use_color`, which
also honors NO_COLOR/FORCE_COLOR) and use it for indentation:

```python
def _is_tty() -> bool:
    return hasattr(sys.stdout, "isatty") and sys.stdout.isatty()

def _pad() -> str:
    return "  " if _is_tty() else ""
```

Change `info`/`success`/`warn` to `print(f"{_pad()}{msg}")` (with their color
wrapping unchanged). `header` keeps its text; bold already gates via `_c`.
`error`/`next_step` stay unindented always — they are structure, not
decoration.

- [x] **Step 4: Run the full suite; fix indentation assertions**

Run: `uv run pytest`
Expected: tests asserting two-space-indented output fail; update them to the
undecorated form (most use substring `in` checks and survive).

- [x] **Step 5: Commit**

```bash
git add src/reins/console.py tests/
git commit -m "feat(console): gate indentation decoration on TTY"
```

### Task 3: `doc_show` module — show with truncation for slug-addressed types

**Files:**
- Create: `src/reins/commands/doc_show.py`
- Test: `tests/commands/test_doc_show.py`

- [x] **Step 1: Write failing tests**

Create `tests/commands/test_doc_show.py`. Use the `kb_repo` fixture plus
`monkeypatch.chdir(kb_repo)`; the fixture's repo scope is `testproject`, and
`repo_slug()` derives from the git remote — check how existing command tests
pin the slug (see `tests/commands/test_doc_create.py`) and copy that setup
exactly. Then:

```python
from reins.commands.doc_show import PREVIEW_CHARS, cmd_doc_list, cmd_doc_show

def _write_spec(kb_repo, slug, body):
    d = kb_repo / "kb" / "testproject" / "specs"
    d.mkdir(parents=True, exist_ok=True)
    (d / f"{slug}.md").write_text(body)

def test_show_short_doc_prints_all(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    _write_spec(kb_repo, "my-spec", "# My Spec\n\n**Status:** draft\n\nBody.\n")
    assert cmd_doc_show("spec", "my-spec") == 0
    out = capsys.readouterr().out
    assert "# My Spec" in out
    assert "truncated" not in out

def test_show_long_doc_truncates_with_hint(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    _write_spec(kb_repo, "big", "# Big\n\n" + "x" * (PREVIEW_CHARS * 2))
    assert cmd_doc_show("spec", "big") == 0
    out = capsys.readouterr().out
    assert "… (truncated," in out
    assert "chars total)" in out
    assert "next: reins spec show big --full" in out
    assert len(out) < PREVIEW_CHARS * 2

def test_show_full_flag_prints_everything(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    body = "# Big\n\n" + "x" * (PREVIEW_CHARS * 2)
    _write_spec(kb_repo, "big", body)
    assert cmd_doc_show("spec", "big", full=True) == 0
    out = capsys.readouterr().out
    assert "truncated" not in out
    assert "x" * (PREVIEW_CHARS * 2) in out

def test_show_unknown_slug_lists_valid_slugs(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    _write_spec(kb_repo, "real-spec", "# Real\n")
    assert cmd_doc_show("spec", "nope") == 1
    out = capsys.readouterr().out
    assert "error: no spec named 'nope'" in out
    assert "real-spec" in out
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_doc_show.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'reins.commands.doc_show'`.

- [x] **Step 3: Implement `src/reins/commands/doc_show.py`**

```python
"""Per-type kb doc reading: show (truncated preview, --full escape hatch) and list."""

from __future__ import annotations

from pathlib import Path

from reins import console
from reins.doc_types import REGISTRY
from reins.git import current_branch, repo_root, repo_slug, sanitize_branch
from reins.kb import require_kb_dir

PREVIEW_CHARS = 1500

def _repo_dir() -> Path | None:
    root = repo_root()
    if root is None:
        return None
    kb_dir = require_kb_dir(root)
    return kb_dir / repo_slug()

def _doc_files(doc_type: str, repo_dir: Path) -> list[Path]:
    """All docs of a slug-addressed type, index files excluded."""
    dt = REGISTRY[doc_type]
    base = repo_dir / dt.dir_path
    if doc_type == "debt":
        files = sorted((base / "by-domain").glob("*.md"))
    elif doc_type == "idea":
        files = sorted(base.glob("*/*.md"))  # ideas/{username}/{slug}.md
    else:  # spec, prd
        files = sorted(base.glob("*.md"))
    return [f for f in files if f.name != "index.md"]

def _print_doc(target: Path, doc_type: str, ref: str, full: bool) -> None:
    text = target.read_text()
    if full or len(text) <= PREVIEW_CHARS:
        print(text.rstrip())
        return
    print(text[:PREVIEW_CHARS].rstrip())
    print(f"… (truncated, {len(text)} chars total)")
    console.next_step(f"reins {doc_type} show {ref} --full")

def cmd_doc_show(doc_type: str, slug: str, full: bool = False) -> int:
    repo_dir = _repo_dir()
    if repo_dir is None:
        return 1
    matches = {f.stem: f for f in _doc_files(doc_type, repo_dir)}
    target = matches.get(slug)
    if target is None:
        console.error(f"no {doc_type} named '{slug}'")
        if matches:
            console.info(f"valid slugs: {', '.join(sorted(matches))}")
        else:
            print(f"{doc_type}s: 0 found")
            console.next_step(f'reins {doc_type} create "<title>"')
        return 1
    _print_doc(target, doc_type, slug, full)
    return 0
```

- [x] **Step 4: Run to verify pass**

Run: `uv run pytest tests/commands/test_doc_show.py -v`
Expected: PASS.

- [x] **Step 5: Commit**

```bash
git add src/reins/commands/doc_show.py tests/commands/test_doc_show.py
git commit -m "feat(show): doc_show module with truncated preview and --full"
```

### Task 4: `list` verbs with minimal schema and definitive empty state

**Files:**
- Modify: `src/reins/commands/doc_show.py`
- Test: `tests/commands/test_doc_show.py`

- [x] **Step 1: Write failing tests**

```python
def test_list_shows_count_slug_title_status(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    _write_spec(kb_repo, "a-spec", "# Alpha Spec\n\n**Status:** approved\n")
    assert cmd_doc_list("spec") == 0
    out = capsys.readouterr().out
    assert "specs: 1 total" in out
    assert "a-spec — Alpha Spec [approved]" in out
    assert "next: reins spec show <slug>" in out

def test_list_empty_is_definitive(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    assert cmd_doc_list("spec") == 0
    out = capsys.readouterr().out
    assert "specs: 0 found" in out
    assert 'next: reins spec create "<title>"' in out
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_doc_show.py -v`
Expected: FAIL — `ImportError: cannot import name 'cmd_doc_list'`.

- [x] **Step 3: Implement — append to `doc_show.py`**

```python
def _title_and_status(path: Path) -> tuple[str, str]:
    """First `# ` heading and `**Status:**` value from the provenance block."""
    title, status = path.stem, ""
    for line in path.read_text().splitlines()[:12]:
        if line.startswith("# ") and title == path.stem:
            title = line[2:].strip()
        elif line.startswith("**Status:**"):
            status = line.removeprefix("**Status:**").strip()
    return title, status

def cmd_doc_list(doc_type: str) -> int:
    repo_dir = _repo_dir()
    if repo_dir is None:
        return 1
    files = _doc_files(doc_type, repo_dir)
    if not files:
        print(f"{doc_type}s: 0 found")
        console.next_step(f'reins {doc_type} create "<title>"')
        return 0
    print(f"{doc_type}s: {len(files)} total")
    for f in files:
        title, status = _title_and_status(f)
        line = f"{f.stem} — {title}"
        if status:
            line += f" [{status}]"
        console.info(line)
    console.next_step(f"reins {doc_type} show <slug>")
    return 0
```

- [x] **Step 4: Run to verify pass**

Run: `uv run pytest tests/commands/test_doc_show.py -v` — expected PASS.

- [x] **Step 5: Commit**

```bash
git add src/reins/commands/doc_show.py tests/commands/test_doc_show.py
git commit -m "feat(show): list verbs with minimal schema and definitive empty state"
```

### Task 5: branch-addressed show (`plan show`, `retro show`)

**Files:**
- Modify: `src/reins/commands/doc_show.py`
- Test: `tests/commands/test_doc_show.py`

- [x] **Step 1: Write failing tests**

```python
from reins.commands.doc_show import cmd_plan_show

def test_plan_show_defaults_to_current_branch(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    # kb_repo starts on 'main'; put a plan there
    plan_dir = kb_repo / "kb" / "testproject" / "exec-plans" / "active" / "main"
    plan_dir.mkdir(parents=True, exist_ok=True)
    (plan_dir / "plan.md").write_text("# Execution Plan: main\n\n## Goal\nShip.\n")
    assert cmd_plan_show() == 0
    out = capsys.readouterr().out
    assert "# Execution Plan: main" in out

def test_plan_show_missing_suggests_create(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    assert cmd_plan_show("no-such-branch") == 1
    out = capsys.readouterr().out
    assert "error: no plan for branch 'no-such-branch'" in out
    assert "next: reins plan create" in out
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_doc_show.py -v`
Expected: FAIL — `ImportError: cannot import name 'cmd_plan_show'`.

- [x] **Step 3: Implement — append to `doc_show.py`**

```python
def _branch_doc_show(doc_type: str, branch: str | None, full: bool) -> int:
    repo_dir = _repo_dir()
    if repo_dir is None:
        return 1
    branch = branch or current_branch()
    if not branch:
        console.error("no branch given and none checked out")
        return 1
    dt = REGISTRY[doc_type]
    safe = sanitize_branch(branch)
    target = repo_dir / dt.dir_path / dt.filename.format(branch=safe)
    if not target.is_file():
        console.error(f"no {doc_type} for branch '{branch}'")
        console.next_step(f"reins {doc_type} create")
        return 1
    _print_doc(target, doc_type, safe, full)
    return 0

def cmd_plan_show(branch: str | None = None, full: bool = False) -> int:
    return _branch_doc_show("plan", branch, full)

def cmd_retro_show(branch: str | None = None, full: bool = False) -> int:
    return _branch_doc_show("retro", branch, full)
```

- [x] **Step 4: Run to verify pass**

Run: `uv run pytest tests/commands/test_doc_show.py -v` — expected PASS.

- [x] **Step 5: Commit**

```bash
git add src/reins/commands/doc_show.py tests/commands/test_doc_show.py
git commit -m "feat(show): plan/retro show addressed by branch"
```

### Task 6: CLI wiring for show/list

**Files:**
- Modify: `src/reins/cli.py`
- Test: `tests/commands/test_dispatch.py`, `tests/test_cli_shape.py`

- [x] **Step 1: Extend the parser in `_build_parser()`**

In `_doc_group_with_create_title` (cli.py:23), after the `create` subparser:

```python
        sp = gs.add_parser("show", help=f"Show a {name} doc (truncated; --full for all)")
        sp.add_argument("slug", help="Doc slug (see 'list')")
        sp.add_argument("--full", action="store_true", help="Print the whole doc")
        gs.add_parser("list", help=f"List {name} docs")
```

Add the same `show`/`list` pair to the `idea` group (after `idea create`).
Add to the `plan` group:

```python
    plan_show_p = plan_sub.add_parser("show", help="Show the plan doc (truncated; --full for all)")
    plan_show_p.add_argument("branch", nargs="?", default=None, help="Branch (default: current)")
    plan_show_p.add_argument("--full", action="store_true", help="Print the whole doc")
```

Add the identical `show` subparser to the `retro` group.

- [x] **Step 2: Add `_DISPATCH` entries**

```python
    ("spec", "show"): lambda a: _load("doc_show", "cmd_doc_show")("spec", a.slug, full=a.full),
    ("spec", "list"): lambda _: _load("doc_show", "cmd_doc_list")("spec"),
    ("prd", "show"): lambda a: _load("doc_show", "cmd_doc_show")("prd", a.slug, full=a.full),
    ("prd", "list"): lambda _: _load("doc_show", "cmd_doc_list")("prd"),
    ("debt", "show"): lambda a: _load("doc_show", "cmd_doc_show")("debt", a.slug, full=a.full),
    ("debt", "list"): lambda _: _load("doc_show", "cmd_doc_list")("debt"),
    ("idea", "show"): lambda a: _load("doc_show", "cmd_doc_show")("idea", a.slug, full=a.full),
    ("idea", "list"): lambda _: _load("doc_show", "cmd_doc_list")("idea"),
    ("plan", "show"): lambda a: _load("doc_show", "cmd_plan_show")(a.branch, full=a.full),
    ("retro", "show"): lambda a: _load("doc_show", "cmd_retro_show")(a.branch, full=a.full),
```

- [x] **Step 3: Run the CLI-shape and dispatch tests**

Run: `uv run pytest tests/test_cli_shape.py tests/commands/test_dispatch.py -v`
Expected: PASS if those tests derive coverage from the parser; if they
enumerate verbs explicitly, add the new `(noun, verb)` pairs following the
file's existing pattern, re-run, PASS.

- [x] **Step 4: End-to-end smoke check**

Run: `uv run reins spec list && uv run reins spec show agent-native-output-surface-axi-principles`
Expected: exit 0; the list includes this spec's slug; show prints a truncated
preview ending in `next: reins spec show agent-native-output-surface-axi-principles --full`.

- [x] **Step 5: Commit**

```bash
git add src/reins/cli.py tests/
git commit -m "feat(cli): wire show/list verbs for all doc types"
```

### Task 7: content-first home view

**Files:**
- Create: `src/reins/commands/home.py`
- Modify: `src/reins/cli.py:236-238` (the no-args branch in `main()`)
- Test: `tests/commands/test_home.py`

- [x] **Step 1: Write failing tests**

Create `tests/commands/test_home.py` (same fixture/slug setup as
`test_doc_show.py`):

```python
from reins.commands.home import cmd_home

def test_home_shows_live_state_not_usage(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    assert cmd_home() == 0
    out = capsys.readouterr().out
    assert "bin: " in out
    assert "branch: main" in out
    assert "plan: none for this branch" in out
    assert "next: reins plan create" in out
    assert "usage:" not in out

def test_home_outside_git_repo_is_definitive(tmp_path, monkeypatch, capsys):
    monkeypatch.chdir(tmp_path)
    assert cmd_home() == 0
    out = capsys.readouterr().out
    assert "repo: not inside a git repository" in out
    assert "next: reins help" in out

def test_bare_reins_invokes_home(kb_repo, monkeypatch, capsys):
    from reins.cli import main
    monkeypatch.chdir(kb_repo)
    assert main([]) == 0
    out = capsys.readouterr().out
    assert "branch: main" in out
    assert "usage:" not in out
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_home.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'reins.commands.home'`.

- [x] **Step 3: Implement `src/reins/commands/home.py`**

```python
"""Bare `reins` — content-first home view (axi principle 8).

Local reads only: no fetch, no network, no per-doc `git log` scans. Bare
`reins` must stay near-instant; it is the orientation command.
"""

from __future__ import annotations

import sys
from pathlib import Path

from reins import __version__, console
from reins.doc_types import REGISTRY
from reins.git import current_branch, repo_root, repo_slug, sanitize_branch
from reins.kb import check_overlap, get_kb_dir

def _bin_path() -> str:
    exe = Path(sys.argv[0]).resolve()
    try:
        return f"~/{exe.relative_to(Path.home())}"
    except ValueError:
        return str(exe)

def cmd_home() -> int:
    print(f"bin: {_bin_path()}")
    print(f"reins {__version__} — agentic engineering knowledgebase CLI")

    root = repo_root(quiet=True)
    if root is None:
        print("repo: not inside a git repository")
        console.next_step("reins help")
        return 0

    kb_dir = get_kb_dir(root)
    if kb_dir is None:
        print("kb: not set up in this repo")
        console.next_step("reins init", "reins help")
        return 0

    branch = current_branch()
    print(f"branch: {branch or 'detached'}")

    repo_dir = kb_dir / repo_slug()
    active = repo_dir / REGISTRY["plan"].dir_path / "active"
    plans = (
        sorted(d.name for d in active.iterdir() if d.is_dir())
        if active.is_dir()
        else []
    )
    print(f"plans: {len(plans)} active in this repo scope")

    current = sanitize_branch(branch) if branch else ""
    if branch:
        check_overlap(branch, root)
    if current and current in plans:
        print(f"plan: {current} (this branch)")
        console.next_step("reins plan show", "reins kb status")
    else:
        print("plan: none for this branch")
        console.next_step("reins plan create", "reins kb status")
    return 0
```

(If `get_kb_dir`'s actual signature in `src/reins/kb.py` differs — e.g. takes
no argument or returns via exception — adapt the call here to match it; do
not change `kb.py`.)

- [x] **Step 4: Rewire `main()` in `src/reins/cli.py`**

Replace:

```python
    if not argv or argv[0] == "help":
        parser.print_help()
        return 0
```

with:

```python
    if not argv:
        from reins.commands.home import cmd_home
        return cmd_home()

    if argv[0] == "help":
        parser.print_help()
        return 0
```

- [x] **Step 5: Run to verify pass, plus timing check**

Run: `uv run pytest tests/commands/test_home.py -v` — expected PASS.
Run: `time uv run reins` — expected well under 1s after interpreter startup;
no network access.

- [x] **Step 6: Commit**

```bash
git add src/reins/commands/home.py src/reins/cli.py tests/commands/test_home.py
git commit -m "feat(home): content-first bare-reins view with live state"
```

### Task 8: `kb status --compact` + session-start wiring

**Files:**
- Modify: `src/reins/commands/status.py`, `src/reins/cli.py` (kb status parser + dispatch), `.claude/hooks/session-start.sh`
- Test: `tests/commands/test_status_compact.py`

- [x] **Step 1: Write failing test**

Create `tests/commands/test_status_compact.py` (same setup pattern):

```python
from reins.commands.status import cmd_status

def test_compact_is_short_and_undecorated(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    assert cmd_status(compact=True) == 0
    out = capsys.readouterr().out
    lines = [ln for ln in out.splitlines() if ln.strip()]
    assert len(lines) <= 10
    assert any(ln.startswith("reins: branch main") for ln in lines)
    assert "next: reins plan create" in out
    assert "Health" not in out  # no headers, no stale scan
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_status_compact.py -v`
Expected: FAIL — `cmd_status() got an unexpected keyword argument 'compact'`.

- [x] **Step 3: Implement in `src/reins/commands/status.py`**

Change the signature to `def cmd_status(compact: bool = False) -> int:` and at
the top, after resolving `root`, `kb_dir`, `branch`, add:

```python
    if compact:
        return _compact_status(root, kb_dir, branch)
```

Then add (imports: `repo_slug` from `reins.git`):

```python
def _compact_status(root, kb_dir, branch) -> int:
    """≤10-line undecorated dashboard for session-start injection.

    No stale scan (per-doc `git log` is too slow for every session) and no
    headers — this output loads into agent context every session.
    """
    repo_dir = kb_dir / repo_slug()
    active = repo_dir / REGISTRY["plan"].dir_path / "active"
    plans = (
        sorted(d.name for d in active.iterdir() if d.is_dir())
        if active.is_dir()
        else []
    )
    current = sanitize_branch(branch) if branch else ""
    has_plan = current in plans
    state = "plan present" if has_plan else "no plan"
    print(f"reins: branch {branch or 'detached'} — {state}")
    print(f"plans: {len(plans)} active in this repo scope")
    if branch:
        check_overlap(branch, root)
    if has_plan:
        console.next_step("reins plan show")
    else:
        console.next_step("reins plan create")
    return 0
```

- [x] **Step 4: Wire the flag in `src/reins/cli.py`**

Replace `kb_sub.add_parser("status", help="Show kb status and health")` with:

```python
    kb_status_p = kb_sub.add_parser("status", help="Show kb status and health")
    kb_status_p.add_argument(
        "--compact", action="store_true",
        help="≤10-line dashboard for session-start context injection",
    )
```

And the dispatch entry:

```python
    ("kb", "status"): lambda a: _load("status", "cmd_status")(
        compact=getattr(a, "compact", False)
    ),
```

- [x] **Step 5: Run to verify pass**

Run: `uv run pytest tests/commands/test_status_compact.py tests/test_cli_shape.py -v` — expected PASS.
Run: `uv run reins kb status --compact` — expected ≤10 lines ending in a `next:` line.

- [x] **Step 6: Wire `session-start.sh`**

In `.claude/hooks/session-start.sh`, insert between the kb-pull block (ends
line 13) and the remote-only guard (line 16):

```bash
# --- Ambient context: compact kb dashboard (axi principle 7) ---
# Stdout from SessionStart is injected into agent context — keep it minimal.
if command -v reins &>/dev/null; then
  reins kb status --compact 2>/dev/null || true
fi
```

Run: `bash .claude/hooks/session-start.sh` from the repo root — expected: the
compact dashboard prints, exit 0. Also run `shellcheck .claude/hooks/session-start.sh`
(the kb lint runs shellcheck; keep it clean).

Note: the spec asks for the same ambient context on other platforms via their
hook mechanisms. `--compact` is platform-neutral and ready for that; wiring
Cursor/Copilot/Gemini session hooks belongs to the `init`/platform-instructions
surface and is deferred to a follow-up (capture with
`reins idea create "wire kb status --compact into non-Claude session hooks"`
during this task).

- [x] **Step 7: Commit**

```bash
git add src/reins/commands/status.py src/reins/cli.py tests/commands/test_status_compact.py .claude/hooks/session-start.sh
git commit -m "feat(status): --compact dashboard wired into session-start ambient context"
```

### Task 9: `next:` footers and progress channel on existing commands

**Files:**
- Modify: `src/reins/commands/doc_create.py:151`, `src/reins/commands/status.py:52-53`, `src/reins/commands/publish.py`
- Test: `tests/commands/test_doc_create.py`

- [x] **Step 1: Write failing test**

Append to `tests/commands/test_doc_create.py` (reuse its existing setup for a
working create call, matching however its current tests invoke
`cmd_spec_create`):

```python
def test_create_suggests_publish(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    assert cmd_spec_create("My Spec") == 0
    out = capsys.readouterr().out
    assert "next: reins kb publish" in out
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_doc_create.py -v`
Expected: the new test FAILS on the missing `next:` line.

- [x] **Step 3: Implement**

In `doc_create.py` `_create_typed`, after `console.success(f"Created: {filepath}")`:

```python
    console.next_step("reins kb publish")
```

In `status.py` (full mode), replace:

```python
            console.warn("Current branch has no execution plan.")
            console.info("Run 'reins plan create' to start one.")
```

with:

```python
            console.warn("Current branch has no execution plan.")
            console.next_step("reins plan create")
```

In `publish.py`, find the in-flight message (`"Publishing kb changes..."`) and
route it through `console.progress(...)` so stdout carries only the result.

- [x] **Step 4: Run affected tests**

Run: `uv run pytest tests/commands/ tests/test_commit_kb.py -v`
Expected: PASS (update any assertion that expected the old
"Run 'reins plan create'" phrasing or publish's progress line on stdout).

- [x] **Step 5: Commit**

```bash
git add src/reins/commands/ tests/
git commit -m "feat(output): next-step footers on create/status, publish progress to stderr"
```

### Task 10: idempotent no-op messaging

**Files:**
- Modify: `src/reins/commands/mode_cmds.py`, `src/reins/commands/hooks_install.py`
- Test: `tests/test_mode.py`

- [x] **Step 1: Write failing tests**

Append to `tests/test_mode.py` (match its existing fixture for mode state —
it isolates `.reins-mode` via a tmp repo):

```python
def test_enable_twice_is_explicit_noop(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    from reins.commands.mode_cmds import cmd_enable
    assert cmd_enable() == 0
    capsys.readouterr()
    assert cmd_enable() == 0
    out = capsys.readouterr().out
    assert "(no-op)" in out
    assert "already enabled" in out
```

- [x] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_mode.py -v` — the new test FAILS (no "(no-op)").

- [x] **Step 3: Implement**

In `mode_cmds.py`:

```python
def cmd_enable() -> int:
    if get_mode() == "enabled":
        console.info("Reins already enabled (no-op).")
        return 0
    set_mode("enabled")
    console.success("Reins enabled. Hooks and publishing are active.")
    return 0

def cmd_disable() -> int:
    if get_mode() == "disabled":
        console.info("Reins already disabled (no-op).")
        return 0
    set_mode("disabled")
    console.success("Reins disabled. All hooks are no-ops, publishing is blocked.")
    console.next_step("reins mode enable")
    return 0
```

Also update `cmd_incognito`'s existing already-incognito message to include
`(no-op)`.

In `hooks_install.py` `cmd_hooks_install`, after the summary block (the
`Installed:/Appended:/Skipped:` lines), when nothing changed:

```python
    if installed == 0 and appended == 0:
        console.info("hooks: already installed (no-op)")
```

Also move the two progress lines (`"Installing reins git hooks..."` and
`"Installing editor hooks..."`) to `console.progress(...)`.

- [x] **Step 4: Run to verify pass, then verify `update` re-run manually**

Run: `uv run pytest tests/test_mode.py tests/test_init_hook.py -v` — expected PASS
(fix any assertion touched by the hooks progress-channel move).
Run: `uv run reins update && uv run reins update; echo "exit=$?"`
Expected: both runs exit 0; second run reports files as up to date / skipped.
If the second run exits non-zero, fix `src/reins/commands/update.py` to
return 0 on the all-current case before proceeding.

- [x] **Step 5: Commit**

```bash
git add src/reins/commands/mode_cmds.py src/reins/commands/hooks_install.py tests/
git commit -m "feat(idempotency): explicit exit-0 no-ops for mode and hooks install"
```

### Task 11: structural enforcement tests + full verification

**Files:**
- Create: `tests/test_output_conventions.py`

- [x] **Step 1: Write the structural tests**

```python
"""Mechanical enforcement of the axi output conventions (spec §6).

Channel model: stdout = data/errors/suggestions, stderr = progress/debug,
exit codes = status. These tests are the lint that spec §6 requires before
the convention can become a golden principle.
"""

from reins import console

def test_error_channel_and_shape(capsys):
    console.error("boom")
    out, err = capsys.readouterr()
    assert out.startswith("error: boom")
    assert err == ""

def test_progress_channel(capsys):
    console.progress("working...")
    out, err = capsys.readouterr()
    assert out == ""
    assert err == "working...\n"

def test_next_step_shape(capsys):
    console.next_step("reins plan create")
    out, err = capsys.readouterr()
    assert out == "next: reins plan create\n"
    assert err == ""

def test_no_direct_stderr_prints_in_commands():
    """Agent-facing modules must use console.* channels, not raw stderr prints."""
    from pathlib import Path
    import reins.commands as commands

    offenders = []
    for py in Path(commands.__path__[0]).rglob("*.py"):
        text = py.read_text()
        if "file=sys.stderr" in text:
            offenders.append(py.name)
    assert offenders == [], f"raw stderr prints in: {offenders}"
```

- [x] **Step 2: Run them; fix any offender the sweep finds**

Run: `uv run pytest tests/test_output_conventions.py -v`
Expected: PASS. If `test_no_direct_stderr_prints_in_commands` fails, route the
offending prints through `console.progress` (diagnostics) or `console.error`
(failures) and re-run.

- [x] **Step 3: Full verification**

Run: `uv run pytest && uv run ruff check && uv run pyright && uv run reins kb lint`
Expected: all pass (kb lint may keep its pre-existing warnings; no new ones).

- [x] **Step 4: Commit**

```bash
git add tests/test_output_conventions.py
git commit -m "test: structural enforcement of axi output conventions"
```

- [x] **Step 5: Add the golden principle (now that the lint exists)**

Run:

```bash
uv run reins principle add "Axi output surface: stdout is for agents"
```

Then edit the appended entry in `kb/reins/golden-principles.md` to:

```markdown
   - Data, errors (`error: …`), and `next:` suggestions go to stdout; stderr
     carries only progress/debug; exit codes carry status (0 incl. no-ops,
     1 error, 2 usage). Enforced by tests/test_output_conventions.py.
   - Prevents: agents misreading progress noise as data, and follow-up
     round trips after ambiguous output.
```

Then: `uv run reins kb publish`

- [x] **Step 6: Final commit + plan progress update**

Update `kb/reins/exec-plans/active/feature-axi-output-surface/progress.md`
with what shipped, then:

```bash
git add -A && git commit -m "docs: record axi output surface completion"
```

## Dependencies

- Spec: `kb/reins/specs/agent-native-output-surface-axi-principles.md` (approved, human-validated)
- No overlap expected with other active branches (verify with `reins plan status`)
