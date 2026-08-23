---
type: plan
title: 'Execution Plan: feat-registry-doc-types-stage2'
slug: feat-registry-doc-types-stage2
lifecycle: active
status: planning
created: 2026-08-16
author: Michael Biehl
branch: feat-registry-doc-types-stage2
ticket: N/A
spec: reinicorn/specs/registry-driven-doc-types.md
---

# Execution Plan: feat-registry-doc-types-stage2

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

## Goal

Stage 2 of the registry-driven-doc-types spec: CLI generation. One loop over `REGISTRY` emits the doc-type parser groups and their `_DISPATCH` rows from `addressing`/`create_verb`/`title_source`/`help_text`. Only plan's `create`/`status`/`complete` verbs remain hand-wired (they route to the bespoke lifecycle commands). Drop the defensive `getattr(a, "include_drafts", False)` defaults.

**Branch flow:** this branch PRs into the long-lived integration branch `feat-registry-doc-types` (never `main`). Stage 1 (fields + generic creator) already merged there.

**Architecture:** `cli.py` gains two small registry-driven generators — `_add_doc_type_groups(sub)` for parser groups and `_doc_dispatch_rows()` for dispatch rows — replacing `_doc_group_with_create_title`, the five hand-written per-type parser blocks, and ~17 hand-written `_DISPATCH` rows. `doc_show.py` gains one generic `cmd_branch_show(doc_type, branch, full)` replacing `cmd_plan_show`/`cmd_retro_show`, so branch-addressed show rows can be generated.

**Tech Stack:** Python 3.12, argparse, uv, pytest.

## Global Constraints

- Behavior-preserving: the full existing suite stays green in every task (help *wording* normalizations are the one disclosed exception, listed in Approach).
- No stringly-typed code (golden principle 15): branch on `Enum` members with `is` (`Addressing`/`TitleSource`/`CreateMode`), registry identity checks (`dt is REGISTRY["retro"]`), never string comparisons. `test_no_doc_type_key_comparisons` enforces the doc-type-key slice.
- All paths and metadata come from `reinicorn.doc_types.REGISTRY` (golden principle P2).
- stdout is the agent-facing result surface; stderr is progress/debug only.
- Red-green TDD; conventional commits; frequent commits.
- Gate per task: `uv run pytest tests/ -v`, `uv run ruff check src/reinicorn tests`, `uv run pyright src/reinicorn`, `bash tests/run-all.sh`.
- Implementer subagents: never push, never run the rcorn CLI, don't touch `kb/`.

## Acceptance Criteria

- [ ] `cli.py` contains no per-doc-type parser blocks or dispatch rows: the doc-type surface (spec/prd/debt/idea/plan/retro/principle groups; create/show/list verbs) is emitted by one loop over `REGISTRY` per generator.
- [ ] Only plan's `create`/`status`/`complete` remain hand-wired (routing to `cmd_plan_create`/`cmd_plan_status`/`cmd_plan_complete`); plan's `show` is generated like retro's.
- [ ] No `getattr(a, "include_drafts", False)` remains; generated rows read `a.include_drafts` directly.
- [ ] Phantom-type test (stage-2 slice of the spec's executable design goal): a synthetic slug-addressed registry row gets a full CLI surface — parser group with create/show/list and working dispatch rows — with no change outside `REGISTRY`.
- [ ] Existing CLI shape and dispatch tests pass (migrated pins included); full gate green.
- [ ] PR to the `feat-registry-doc-types` integration branch; PR body cites the spec, states the gate evidence, and discloses the help-wording normalizations.

## Approach

Two generators in `cli.py`, both iterating `REGISTRY.values()`:

1. `_add_doc_type_groups(sub)` — called from `_build_parser()`. Per `DocType`: a group parser named `dt.key` with `dt.help_text`; a create verb named `dt.create_verb` whose positional derives from `title_source` (TITLE → `title` nargs="+"; FREE_TEXT → `text` nargs="+"; NONE → no positional); for `Addressing.SLUG`: `show` (slug, `--full`, `--include-drafts`) and `list` (`--include-drafts`); for `Addressing.BRANCH`: `show` (optional branch, `--full`); for `Addressing.SINGLETON`: no show/list (today's principle surface). After the loop, plan's `status` and `complete` verbs are appended hand-wired.
2. `_doc_dispatch_rows()` — returns the generated `(key, verb) → handler` dict; module-level `_DISPATCH = {**_doc_dispatch_rows(), ...}` with the hand-wired rows (plan create/status/complete, review, kb, mode, init, hooks, update, feedback) merged after, so plan's create override wins.

`doc_show.py`: `cmd_plan_show`/`cmd_retro_show` collapse into `cmd_branch_show(doc_type, branch=None, full=False)`. Retro's rides-with-active-plan lookup stays inside it behind `dt is REGISTRY["retro"]` — the same sanctioned pattern as `_branch_target` in `doc_create.py` (spec non-goal: that coupling stays code).

**Disclosed normalizations** (cosmetic, help output only — no command behavior changes):

1. Verb help strings become formulaic: TITLE create → `Create a {key} doc`; FREE_TEXT create → `Capture free-form {key} text` (idea's was "Capture an idea"); NONE create → `Create the {key} for the current branch` (plan's was "Create execution plan for current branch", retro's "Create retro for current branch"); APPEND → `Append a {key}` (principle's was "Append a principle" — unchanged); SLUG show → `Show a {key} doc (truncated; --full for all)`; BRANCH show → `Show the {key} doc (truncated; --full for all)`; list → `List {key} docs`.
2. Verb order in `plan --help` becomes create, show, status, complete (loop emits create/show; hand-wired verbs append). Shape tests assert presence, not order.
3. `cmd_plan_show`/`cmd_retro_show` are deleted in favor of `cmd_branch_show`; the two dispatch pin tests migrate to the new handler.
4. Free-text positional stays named `text` (idea's current name), title positional stays `title` — both derived from `title_source`, so `a.text`/`a.title` reads stay valid.

## Tasks

### Task 1: Generic branch-addressed show (`cmd_branch_show`)

**Files:**
- Modify: `src/reinicorn/commands/doc_show.py` (lines 159–185: replace `cmd_plan_show` and `cmd_retro_show`)
- Modify: `src/reinicorn/cli.py` (dispatch rows for `("plan", "show")` and `("retro", "show")`)
- Test: `tests/commands/test_dispatch.py`, `tests/commands/test_doc_show.py`

**Interfaces:**
- Consumes: `_branch_doc_show(doc_type, branch, full)`, `_print_doc`, `_missing_branch_doc`, `_branch_doc_pattern` (all existing in `doc_show.py`); `REGISTRY`, `branch_doc_path`, `branch_dir_name`.
- Produces: `cmd_branch_show(doc_type: str, branch: str | None = None, full: bool = False) -> int` — Task 3's generated BRANCH show rows call exactly this.

- [ ] **Step 1: Migrate the two dispatch pin tests (red)**

In `tests/commands/test_dispatch.py`, replace `test_plan_show_dispatches_to_cmd_plan_show` and `test_retro_show_dispatches_to_cmd_retro_show` with:

```python
@pytest.mark.parametrize("noun,argv_tail,expected_branch", [
    ("plan", ["show", "some-branch"], "some-branch"),
    ("retro", ["show"], None),
])
def test_branch_show_dispatches_to_cmd_branch_show(noun, argv_tail, expected_branch):
    with patch(
        "reinicorn.commands.doc_show.cmd_branch_show", return_value=0
    ) as mock_show:
        assert main([noun, *argv_tail]) == 0
    mock_show.assert_called_once_with(noun, expected_branch, full=False)
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_dispatch.py -v -k branch_show`
Expected: FAIL (AttributeError: `cmd_branch_show` does not exist)

- [ ] **Step 3: Implement `cmd_branch_show`, delete the wrappers**

In `src/reinicorn/commands/doc_show.py`, replace `cmd_plan_show` and `cmd_retro_show` (lines 159–185) with:

```python
def cmd_branch_show(
    doc_type: str, branch: str | None = None, full: bool = False,
) -> int:
    """Show a branch-addressed doc. Retro checks the active plan dir first
    (retro rides with an active plan until archive; identity check against
    the registry row keeps type knowledge out of string comparisons)."""
    dt = REGISTRY[doc_type]
    if dt is not REGISTRY["retro"]:
        return _branch_doc_show(doc_type, branch, full)
    repo_dir = _repo_dir()
    if repo_dir is None:
        return 1
    branch = branch or current_branch()
    if not branch:
        console.error("no branch given and none checked out")
        return 1
    active = branch_doc_path("plan", repo_dir, branch).parent / "retro.md"
    target = active if active.is_file() else branch_doc_path(doc_type, repo_dir, branch)
    if not target.is_file():
        exec_plans = repo_dir / dt.dir_path
        active_pattern = str(
            PurePosixPath(_branch_doc_pattern("plan")).with_name("retro.md")
        )
        branches = {
            f.parent.name
            for pattern in (_branch_doc_pattern(doc_type), active_pattern)
            for f in exec_plans.glob(pattern)
        }
        return _missing_branch_doc(doc_type, branch, branches)
    _print_doc(target, doc_type, branch_dir_name(branch), full)
    return 0
```

In `src/reinicorn/cli.py`, update the two rows:

```python
    ("plan", "show"): lambda a: _load("doc_show", "cmd_branch_show")(
        "plan", a.branch, full=a.full
    ),
    ("retro", "show"): lambda a: _load("doc_show", "cmd_branch_show")(
        "retro", a.branch, full=a.full
    ),
```

- [ ] **Step 4: Migrate direct callers in tests**

Run: `grep -rn "cmd_plan_show\|cmd_retro_show" tests/` and update every hit to call `cmd_branch_show("plan", ...)` / `cmd_branch_show("retro", ...)` with the same arguments and expectations (the underlying behavior is unchanged, so only the callable and the leading `doc_type` argument change).

- [ ] **Step 5: Run the full gate**

Run: `uv run pytest tests/ -v && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && bash tests/run-all.sh`
Expected: all green.

- [ ] **Step 6: Commit**

```bash
git add src/reinicorn/commands/doc_show.py src/reinicorn/cli.py tests/
git commit -m "refactor: generic cmd_branch_show replaces cmd_plan_show/cmd_retro_show"
```

### Task 2: Generate the doc-type parser groups

**Files:**
- Modify: `src/reinicorn/cli.py` (lines 23–101: replace `_doc_group_with_create_title` and the five per-type blocks)
- Test: `tests/test_cli_shape.py`, `tests/commands/test_cli_generation.py` (create)

**Interfaces:**
- Consumes: `REGISTRY`, `Addressing`, `TitleSource` from `reinicorn.doc_types` (import at module top of `cli.py` — `doc_types` has no heavy imports, CLI startup stays cheap).
- Produces: `_add_doc_type_groups(sub: argparse._SubParsersAction) -> None`; positional names `title` (TITLE), `text` (FREE_TEXT), optional `branch` on BRANCH show — Task 3's rows read exactly these attribute names.

- [ ] **Step 1: Write the phantom-surface test (red)**

Create `tests/commands/test_cli_generation.py`:

```python
"""Registry-driven CLI generation: a registry row is the only source of a
doc type's parser surface (spec: registry-driven-doc-types, stage 2)."""
from __future__ import annotations

from unittest.mock import patch

import pytest

from reinicorn.cli import _build_parser
from reinicorn.doc_types import REGISTRY, Addressing, DocType

PHANTOM = DocType(
    key="phantom", dir_path="phantoms", filename="{slug}.md",
    protected=True, create_hint='rcorn phantom create "<title>"',
    help_text="Phantom doc operations",
    template_body="\n## Body\n\n_Text._\n",
    addressing=Addressing.SLUG,
)


def test_phantom_row_gets_parser_surface():
    with patch.dict(REGISTRY, {"phantom": PHANTOM}):
        parser = _build_parser()
        args = parser.parse_args(["phantom", "create", "My", "Doc"])
        assert args.title == ["My", "Doc"]
        args = parser.parse_args(["phantom", "show", "my-doc", "--include-drafts"])
        assert args.slug == "my-doc" and args.include_drafts is True
        args = parser.parse_args(["phantom", "list"])
        assert args.include_drafts is False


def test_phantom_row_absent_without_patch():
    with pytest.raises(SystemExit):
        _build_parser().parse_args(["phantom", "list"])
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_cli_generation.py -v`
Expected: `test_phantom_row_gets_parser_surface` FAILS (SystemExit: invalid choice 'phantom'); the absence test passes.

- [ ] **Step 3: Implement `_add_doc_type_groups`**

In `src/reinicorn/cli.py`, add `from reinicorn.doc_types import REGISTRY, Addressing, TitleSource` to the module imports, then replace `_doc_group_with_create_title`, its three call sites, and the idea/plan/retro/principle parser blocks (keeping plan's `status`/`complete` additions, moved below the loop) with:

```python
    # ── Doc-type groups (generated from the registry; spec:
    # registry-driven-doc-types stage 2) ─────────────────────
    def _add_doc_type_groups(sub) -> None:
        for dt in REGISTRY.values():
            g = sub.add_parser(dt.key, help=dt.help_text)
            gs = g.add_subparsers(dest=f"{dt.key}_command")
            gs.required = True
            if dt.title_source is TitleSource.TITLE:
                cp = gs.add_parser(
                    dt.create_verb,
                    help=(f"Append a {dt.key}" if dt.create_verb == "add"
                          else f"Create a {dt.key} doc"),
                )
                cp.add_argument("title", nargs="+", help="Document title")
            elif dt.title_source is TitleSource.FREE_TEXT:
                cp = gs.add_parser(
                    dt.create_verb, help=f"Capture free-form {dt.key} text"
                )
                cp.add_argument("text", nargs="+", help=f"{dt.key} text")
            else:
                gs.add_parser(
                    dt.create_verb,
                    help=f"Create the {dt.key} for the current branch",
                )
            if dt.addressing is Addressing.SLUG:
                sp = gs.add_parser(
                    "show",
                    help=f"Show a {dt.key} doc (truncated; --full for all)",
                )
                sp.add_argument("slug", help="Doc slug (see 'list')")
                sp.add_argument("--full", action="store_true", help="Print the whole doc")
                sp.add_argument(
                    "--include-drafts", action="store_true",
                    help="Include drafts/ (unapproved) docs",
                )
                lp = gs.add_parser("list", help=f"List {dt.key} docs")
                lp.add_argument(
                    "--include-drafts", action="store_true",
                    help="Include drafts/ (unapproved) docs",
                )
            elif dt.addressing is Addressing.BRANCH:
                sp = gs.add_parser(
                    "show", help=f"Show the {dt.key} doc (truncated; --full for all)"
                )
                sp.add_argument(
                    "branch", nargs="?", default=None,
                    help="Branch name (default: current)",
                )
                sp.add_argument("--full", action="store_true", help="Print the whole doc")
            # Addressing.SINGLETON: create verb only (principle today).

    _add_doc_type_groups(sub)

    # Plan lifecycle verbs stay hand-wired (spec non-goal: plan lifecycle
    # stays code). Fetch the generated group and append them.
    plan_sub = None
    for action in sub.choices["plan"]._actions:
        if isinstance(action, argparse._SubParsersAction):
            plan_sub = action
            break
    assert plan_sub is not None
    plan_sub.add_parser("status", help="Show plan status for current branch")
    plan_complete_p = plan_sub.add_parser("complete", help="Archive plan to completed/")
    plan_complete_p.add_argument(
        "branch", nargs="?", default=None, help="Branch name (default: current)"
    )
```

- [ ] **Step 4: Run the new test and the shape tests**

Run: `uv run pytest tests/commands/test_cli_generation.py tests/test_cli_shape.py -v`
Expected: all PASS.

- [ ] **Step 5: Run the full gate**

Run: `uv run pytest tests/ -v && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && bash tests/run-all.sh`
Expected: all green (dispatch table is still the hand-written one and still matches the generated surface, so `test_every_parser_verb_has_a_dispatch_entry` stays green).

- [ ] **Step 6: Commit**

```bash
git add src/reinicorn/cli.py tests/commands/test_cli_generation.py
git commit -m "refactor: generate doc-type parser groups from the registry"
```

### Task 3: Generate the dispatch rows

**Files:**
- Modify: `src/reinicorn/cli.py` (`_DISPATCH`: replace the 17 doc-type rows; drop `include_drafts` getattr defaults)
- Test: `tests/commands/test_cli_generation.py` (extend), `tests/commands/test_dispatch.py` (unchanged — pins must keep passing)

**Interfaces:**
- Consumes: `_add_doc_type_groups`'s positional names (`title`/`text`/`slug`/`branch`), `_load`, `REGISTRY`, `Addressing`, `TitleSource`; `cmd_doc_create(doc_type, title="")`, `cmd_doc_show(doc_type, slug, full=, include_drafts=)`, `cmd_doc_list(doc_type, include_drafts=)`, `cmd_branch_show(doc_type, branch, full=)` (Task 1).
- Produces: `_doc_dispatch_rows() -> dict`; `_DISPATCH = {**_doc_dispatch_rows(), <hand-wired rows>}`.

- [ ] **Step 1: Write the phantom end-to-end test (red)**

Append to `tests/commands/test_cli_generation.py`:

```python
def test_phantom_row_gets_dispatch_rows():
    from reinicorn.cli import _doc_dispatch_rows
    with patch.dict(REGISTRY, {"phantom": PHANTOM}):
        rows = _doc_dispatch_rows()
    for verb in ("create", "show", "list"):
        assert ("phantom", verb) in rows


def test_phantom_create_end_to_end(kb_repo):
    """Parser + generated row + generic creator: the registry row alone
    yields a working `phantom create` (spec's executable design goal)."""
    from reinicorn.cli import _doc_dispatch_rows
    from tests.commands.test_doc_create import _create_env
    p1, p2, p3, p4 = _create_env(kb_repo)
    with patch.dict(REGISTRY, {"phantom": PHANTOM}), p1, p2, p3, p4:
        args = _build_parser().parse_args(["phantom", "create", "A", "Test", "Doc"])
        assert _doc_dispatch_rows()[("phantom", "create")](args) == 0
    assert (kb_repo / "kb" / "testproject" / "phantoms" / "a-test-doc.md").is_file()
```

(`kb_repo` and `_create_env` already exist in `tests/commands/test_doc_create.py` / conftest from stage 1 — reuse, don't duplicate. If `_create_env` is module-private there, import it anyway; tests importing test helpers from sibling test modules is the established pattern in this suite — check with `grep -rn "_create_env" tests/` and follow whatever the stage-1 tests do.)

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_cli_generation.py -v`
Expected: new tests FAIL (ImportError: `_doc_dispatch_rows` does not exist).

- [ ] **Step 3: Implement `_doc_dispatch_rows`**

In `src/reinicorn/cli.py`, above `_DISPATCH`, add:

```python
def _doc_dispatch_rows() -> dict:
    """Generated (noun, verb) rows for every registry doc type (spec:
    registry-driven-doc-types stage 2). Hand-wired rows merged into
    _DISPATCH after this override plan's create/status/complete."""
    rows: dict = {}
    for dt in REGISTRY.values():
        key = dt.key
        if dt.title_source is TitleSource.TITLE:
            rows[(key, dt.create_verb)] = (
                lambda a, k=key: _load("doc_create", "cmd_doc_create")(
                    k, " ".join(a.title)
                )
            )
        elif dt.title_source is TitleSource.FREE_TEXT:
            rows[(key, dt.create_verb)] = (
                lambda a, k=key: _load("doc_create", "cmd_doc_create")(
                    k, " ".join(a.text)
                )
            )
        else:
            rows[(key, dt.create_verb)] = (
                lambda _, k=key: _load("doc_create", "cmd_doc_create")(k)
            )
        if dt.addressing is Addressing.SLUG:
            rows[(key, "show")] = (
                lambda a, k=key: _load("doc_show", "cmd_doc_show")(
                    k, a.slug, full=a.full, include_drafts=a.include_drafts
                )
            )
            rows[(key, "list")] = (
                lambda a, k=key: _load("doc_show", "cmd_doc_list")(
                    k, include_drafts=a.include_drafts
                )
            )
        elif dt.addressing is Addressing.BRANCH:
            rows[(key, "show")] = (
                lambda a, k=key: _load("doc_show", "cmd_branch_show")(
                    k, a.branch, full=a.full
                )
            )
    return rows
```

Then replace the 17 hand-written doc-type rows in `_DISPATCH` (all spec/prd/debt/idea/retro/principle rows, plan's show row) so the table reads:

```python
_DISPATCH = {
    **_doc_dispatch_rows(),
    # Plan lifecycle verbs stay hand-wired; create overrides the generated
    # row (cmd_doc_create refuses "plan" — lifecycle logic lives in plan.py).
    ("plan", "create"): lambda _: _load("plan", "cmd_plan_create")(),
    ("plan", "status"): lambda _: _load("plan", "cmd_plan_status")(),
    ("plan", "complete"): lambda a: _load("plan", "cmd_plan_complete")(a.branch),
    ("review", "start"): ...,   # ← all remaining hand-wired rows unchanged,
    ...                         #   from ("review", "start") through
}                               #   ("feedback", None)
```

(The `...` above means: keep the existing review/kb/mode/init/hooks/update/feedback rows byte-for-byte as they are today — do not retype them.)

- [ ] **Step 4: Run the generation, dispatch, and shape tests**

Run: `uv run pytest tests/commands/test_cli_generation.py tests/commands/test_dispatch.py tests/test_cli_shape.py -v`
Expected: all PASS — the existing pins (`test_create_dispatches_to_cmd_doc_create`, `test_retro_create_dispatches_without_title`, `test_plan_create_dispatches_to_cmd_plan_create`, `test_branch_show_dispatches_to_cmd_branch_show`, include-drafts pins) now exercise the generated rows.

- [ ] **Step 5: Verify no getattr defaults remain**

Run: `grep -n 'getattr(a, "include_drafts"' src/reinicorn/cli.py`
Expected: no matches.

- [ ] **Step 6: Run the full gate**

Run: `uv run pytest tests/ -v && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && bash tests/run-all.sh`
Expected: all green.

- [ ] **Step 7: Commit**

```bash
git add src/reinicorn/cli.py tests/commands/test_cli_generation.py
git commit -m "refactor: generate doc-type dispatch rows from the registry"
```

### Task 4: Finish — final review and PR

- [ ] **Step 1: Full gate on the branch**

Run: `uv run pytest tests/ -v && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && bash tests/run-all.sh`
Expected: all green.

- [ ] **Step 2: Final whole-branch review** (subagent-driven-development: dispatch the final code reviewer with the branch diff from `git merge-base feat-registry-doc-types HEAD`).

- [ ] **Step 3: Open the PR against the integration branch**

```bash
gh pr create --base feat-registry-doc-types \
  --title "refactor: registry-driven doc types, stage 2 — CLI generation" \
  --body "<cites the spec; gate evidence; discloses the help-wording normalizations from the plan's Approach section>"
```

## Dependencies

- Builds on stage 1 (merged into `feat-registry-doc-types` via PR #50): `DocType` fields, `Addressing`/`TitleSource`/`CreateMode` enums, `cmd_doc_create`.
- Stages 3–5 (gate generalization/literal sweep, seeding+linting, shipped docs/tests) branch off `feat-registry-doc-types` after this merges. The integration branch merges to `main` only when stage 5 lands.
