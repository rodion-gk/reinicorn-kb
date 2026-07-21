# Honest init platform prompt Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Ticket:** N/A
**Author:** Rodion Izotov
**Created:** 2026-07-21
**Status:** planning
**Spec:** `kb/reinicorn/specs/honest-init-platform-prompt.md` (**Status:** approved)

**Goal:** Make `rcorn init`'s platform prompt an honest typed select-set (no fake checkboxes, no toggle), warn on discarded input, and add `--platforms` so init can skip the prompt non-interactively.

**Architecture:** Keep all logic in `src/reinicorn/commands/init.py` (zero new deps). Extract a shared `PLATFORM_OPTIONS` table plus a split-then-strip-per-token helper used by both the interactive number parser and `--platforms` key parsing. CLI passes `platforms_raw: str | None` into `cmd_init`, which validates via `_parse_platforms_flag` and threads `platforms: list[str] | None` into `_setup_assets` (`None` prompts; a list including empty installs without prompting).

**Tech Stack:** Python 3.12+, stdlib only, `pytest` + `unittest.mock`, existing `reinicorn.console`.

## Global Constraints

- Zero new runtime dependencies (no `questionary` / `rich` / TUI libs).
- Interactive model is **select-set**, not toggle; prompt copy must not contain `toggle`.
- Parse order pinned: strip whole input → split on commas → strip each token (never `replace(" ", "")` before split).
- Discarded interactive tokens warn via `console.warn`; no re-prompt loop.
- Flag name is `--platforms` only (no `--harnesses` alias).
- Unknown `--platforms` keys hard-fail at the boundary (`console.error`, non-zero exit).
- Do not change `_install_platform_instructions` behavior for a given key list.
- Do not rewrite `_prompt_kb_source` / gh-auth prompts.
- Existing init tests that mock `_prompt_platforms` stay untouched.

---

## Acceptance Criteria

- [ ] Interactive prompt shows a plain numbered list (no `[x]` / `[ ]`) and select/default wording.
- [ ] Enter keeps defaults (`["claude"]`); typed numbers replace defaults (select-set).
- [ ] Discarded tokens warn; empty Enter does not warn.
- [ ] `rcorn init --platforms cursor` skips the prompt and installs Cursor instructions.
- [ ] `rcorn init --platforms nope` exits non-zero with an error naming the unknown key.
- [ ] All §4 / §5 test cases from the spec are covered; full suite green when the user runs it.

## Approach

Two TDD commits:

1. **Prompt + parser** — `PLATFORM_OPTIONS`, `_split_comma_tokens`, rewrite `_prompt_platforms`, new unit tests in `tests/test_prompt_platforms.py`.
2. **`--platforms` flag** — CLI arg, boundary validation, plumb through `cmd_init` / `_setup_assets`, tests in `tests/test_init_platforms.py`.

Shared comma-token helper is introduced in Task 1 and reused in Task 2 for flag key splitting.

## File map

| File | Responsibility |
|------|----------------|
| `src/reinicorn/commands/init.py` | `PLATFORM_OPTIONS`, parsers, `_prompt_platforms`, `cmd_init` / `_setup_assets` plumbing |
| `src/reinicorn/cli.py` | `--platforms` argparse + dispatch into `cmd_init` |
| `tests/test_prompt_platforms.py` | Unit tests for interactive prompt/parser (new) |
| `tests/test_init_platforms.py` | `--platforms` init integration tests (extend) |

## Tasks

### Task 1: Honest select-set `_prompt_platforms`

**Files:**
- Modify: `src/reinicorn/commands/init.py` (near `PLATFORM_FILES` / `_prompt_platforms`)
- Create: `tests/test_prompt_platforms.py`

**Interfaces:**
- Consumes: `reinicorn.console.warn`; existing `PLATFORM_FILES` / install path unchanged
- Produces:
  - `PLATFORM_OPTIONS: list[tuple[str, str, bool]]` — `(key, label, default)` in display order
  - `_split_comma_tokens(raw: str) -> list[str]` — strip whole string, split on `,`, strip each token, drop empties
  - `_default_platform_keys() -> list[str]` — keys where `default` is True
  - `_prompt_platforms() -> list[str]` — new UX + select-set parse (same return type as today)

- [ ] **Step 1: Write the failing tests**

Create `tests/test_prompt_platforms.py`:

```python
"""Unit tests for honest select-set platform prompt (spec: honest-init-platform-prompt)."""

from __future__ import annotations

from unittest.mock import patch

from reinicorn.commands import init as init_mod


def _run(user_input: str, capsys) -> tuple[list[str], str]:
    with patch("builtins.input", return_value=user_input):
        result = init_mod._prompt_platforms()
    captured = capsys.readouterr()
    # console.warn/print both go to stdout today
    return result, captured.out


def test_prompt_has_no_checkbox_markers(capsys):
    result, out = _run("", capsys)
    assert result == ["claude"]
    assert "[x]" not in out
    assert "[ ]" not in out


def test_prompt_uses_select_wording_not_toggle(capsys):
    _, out = _run("", capsys)
    lower = out.lower()
    assert "select" in lower
    assert "default" in lower
    assert "toggle" not in lower


def test_empty_enter_defaults_no_warning(capsys):
    result, out = _run("", capsys)
    assert result == ["claude"]
    assert "discard" not in out.lower()
    assert "ignored" not in out.lower()


def test_select_2_is_cursor_only(capsys):
    result, _ = _run("2", capsys)
    assert result == ["cursor"]


def test_select_1_2(capsys):
    result, _ = _run("1,2", capsys)
    assert result == ["claude", "cursor"]


def test_out_of_range_9_defaults_with_warning(capsys):
    result, out = _run("9", capsys)
    assert result == ["claude"]
    assert "9" in out


def test_dedup_2_2(capsys):
    result, _ = _run("2,2", capsys)
    assert result == ["cursor"]


def test_order_2_1_is_option_list_order(capsys):
    result, _ = _run("2,1", capsys)
    assert result == ["claude", "cursor"]


def test_abc_defaults_with_warning(capsys):
    result, out = _run("abc", capsys)
    assert result == ["claude"]
    assert "abc" in out


def test_1_abc_keeps_claude_warns(capsys):
    result, out = _run("1,abc", capsys)
    assert result == ["claude"]
    assert "abc" in out


def test_2_comma_space_3(capsys):
    result, out = _run("2, 3", capsys)
    assert result == ["cursor", "copilot"]
    assert "discard" not in out.lower()
    assert "ignored" not in out.lower()


def test_2_space_3_not_fused_to_23(capsys):
    result, out = _run("2 3", capsys)
    assert result == ["claude"]
    assert "2 3" in out
```

- [ ] **Step 2: Run tests to verify they fail**

Ask the user to run (do not run builds/tests yourself unless they ask):

```bash
uv run pytest tests/test_prompt_platforms.py -v
```

Expected: FAIL (old prompt still has `[x]` / `Toggle`, and `"2"` still toggles to `["claude", "cursor"]`).

- [ ] **Step 3: Write minimal implementation**

In `src/reinicorn/commands/init.py`, near `PLATFORM_FILES`, add:

```python
PLATFORM_OPTIONS: list[tuple[str, str, bool]] = [
    ("claude", "Claude Code", True),
    ("cursor", "Cursor", False),
    ("copilot", "GitHub Copilot", False),
    ("codex", "Codex", False),
]


def _split_comma_tokens(raw: str) -> list[str]:
    """Split on commas first, then strip each token; drop empty tokens."""
    return [token.strip() for token in raw.strip().split(",") if token.strip()]


def _default_platform_keys() -> list[str]:
    return [key for key, _label, default in PLATFORM_OPTIONS if default]
```

Replace `_prompt_platforms` with:

```python
def _prompt_platforms() -> list[str]:
    """Interactive select-set for AI coding platforms."""
    print("Which AI coding platforms do you use?")
    print()
    for i, (_key, label, _default) in enumerate(PLATFORM_OPTIONS, 1):
        print(f"  {i}) {label}")
    print()
    default_labels = ", ".join(
        label for _key, label, default in PLATFORM_OPTIONS if default
    )
    raw = input(
        f"Enter numbers to select (e.g. 1,2), or Enter for default [{default_labels}]: "
    )
    stripped = raw.strip()
    if not stripped:
        return _default_platform_keys()

    selected: set[int] = set()
    discarded: list[str] = []
    for token in _split_comma_tokens(stripped):
        if token.isdigit():
            idx = int(token) - 1
            if 0 <= idx < len(PLATFORM_OPTIONS):
                selected.add(idx)
                continue
        discarded.append(token)

    if discarded:
        kept_keys = (
            [PLATFORM_OPTIONS[i][0] for i in range(len(PLATFORM_OPTIONS)) if i in selected]
            if selected
            else _default_platform_keys()
        )
        kept_labels = ", ".join(
            label
            for key, label, _d in PLATFORM_OPTIONS
            if key in kept_keys
        )
        discarded_disp = ", ".join(repr(t) for t in discarded)
        if selected:
            console.warn(
                f"Ignored invalid platform selection token(s) {discarded_disp}; "
                f"using {kept_labels}."
            )
        else:
            console.warn(
                f"Ignored invalid platform selection {discarded_disp}; "
                f"using default [{kept_labels}]."
            )

    if not selected:
        return _default_platform_keys()
    return [
        key for i, (key, _label, _default) in enumerate(PLATFORM_OPTIONS) if i in selected
    ]
```

Notes for the implementer:

- Warning text must include the discarded token text so tests finding `"9"` / `"abc"` / `"2 3"` pass; prefer `repr` so internal spaces stay visible.
- `console.warn` currently prints to stdout — that is fine; tests use `capsys` stdout.
- Do not change `_install_platform_instructions`.

- [ ] **Step 4: Run tests to verify they pass**

Ask the user to run:

```bash
uv run pytest tests/test_prompt_platforms.py -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/reinicorn/commands/init.py tests/test_prompt_platforms.py
git commit -m "$(cat <<'EOF'
feat(init): honest select-set platform prompt

EOF
)"
```

### Task 2: `--platforms` non-interactive flag

**Files:**
- Modify: `src/reinicorn/cli.py` (init parser + `_DISPATCH` for `init`)
- Modify: `src/reinicorn/commands/init.py` (`cmd_init`, `_setup_assets`, `_parse_platforms_flag`)
- Modify: `tests/test_init_platforms.py` (add flag tests only; leave existing mock-based tests unchanged)

**Interfaces:**
- Consumes: `PLATFORM_OPTIONS`, `_split_comma_tokens` from Task 1
- Produces:
  - `_parse_platforms_flag(value: str) -> list[str] | None` — key list on success; `None` after `console.error` on unknown keys
  - `cmd_init(..., platforms_raw: str | None = None)` — when set, parse before asset setup; return `1` if parse fails
  - `_setup_assets(..., platforms: list[str] | None = None)` — `None` → `_prompt_platforms()`; list (including `[]`) → install without prompting

- [ ] **Step 1: Write the failing tests**

Append to `tests/test_init_platforms.py`:

```python
def test_init_platforms_raw_skips_prompt_and_installs_cursor(tmp_path: Path):
    repo = tmp_path / "my-repo"
    _init_repo(repo)

    with patch("reinicorn.commands.init.cmd_hooks_install", return_value=0), \
         patch("reinicorn.commands.init.repo_slug", return_value="my-repo"), \
         patch("reinicorn.commands.init._prompt_platforms") as prompt:
        rc = cmd_init(
            kb_url="unused", local=True, cwd=repo, platforms_raw="cursor"
        )

    assert rc == 0
    prompt.assert_not_called()
    assert (repo / ".cursor" / "rules" / "reinicorn.mdc").is_file()
    assert not (repo / "CLAUDE.md").exists()


def test_init_platforms_raw_unknown_key_errors(tmp_path: Path, capsys):
    repo = tmp_path / "my-repo"
    _init_repo(repo)

    with patch("reinicorn.commands.init.cmd_hooks_install", return_value=0), \
         patch("reinicorn.commands.init.repo_slug", return_value="my-repo"), \
         patch("reinicorn.commands.init._prompt_platforms") as prompt:
        rc = cmd_init(
            kb_url="unused", local=True, cwd=repo, platforms_raw="nope"
        )

    assert rc == 1
    prompt.assert_not_called()
    err = capsys.readouterr().out
    assert "nope" in err
    assert "claude" in err


def test_parse_platforms_flag_dedup_and_order():
    from reinicorn.commands.init import _parse_platforms_flag

    assert _parse_platforms_flag("codex, claude,claude") == ["claude", "codex"]


def test_parse_platforms_flag_empty_string():
    from reinicorn.commands.init import _parse_platforms_flag

    assert _parse_platforms_flag("") == []


def test_cli_accepts_platforms_flag():
    from reinicorn.cli import _build_parser

    args = _build_parser().parse_args(
        ["init", "--local", "--platforms", "cursor,claude"]
    )
    assert args.platforms == "cursor,claude"
```

- [ ] **Step 2: Run tests to verify they fail**

Ask the user to run:

```bash
uv run pytest tests/test_init_platforms.py -v -k "platforms_raw or parse_platforms or accepts_platforms"
```

Expected: FAIL (`platforms_raw` / `_parse_platforms_flag` / `--platforms` missing).

- [ ] **Step 3: Write minimal implementation**

In `init.py`:

```python
def _parse_platforms_flag(value: str) -> list[str] | None:
    """Parse --platforms KEYS. Returns key list, or None on hard error."""
    if value.strip() == "":
        return []
    known = {key for key, _label, _default in PLATFORM_OPTIONS}
    selected: set[str] = set()
    for token in _split_comma_tokens(value):
        if token not in known:
            known_list = ", ".join(key for key, _label, _default in PLATFORM_OPTIONS)
            console.error(
                f"Unknown platform '{token}'. "
                f"Known platforms: {known_list}. "
                f"How to fix: pass a comma-separated subset, "
                f"e.g. --platforms claude,cursor"
            )
            return None
        selected.add(token)
    return [key for key, _label, _default in PLATFORM_OPTIONS if key in selected]
```

Extend `cmd_init`:

```python
def cmd_init(
    *,
    kb_url: str | None = None,
    local: bool = False,
    create_remote: bool = False,
    kb_name: str | None = None,
    cwd: Path | None = None,
    slug: str | None = None,
    platforms_raw: str | None = None,
) -> int:
```

Before each `_setup_assets(...)` call (full-init and existing-kb-without-manifest paths):

```python
    platforms: list[str] | None = None
    if platforms_raw is not None:
        platforms = _parse_platforms_flag(platforms_raw)
        if platforms is None:
            return 1
```

Pass `platforms=platforms` into `_setup_assets`.

Update `_setup_assets`:

```python
def _setup_assets(
    r_root: Path,
    cwd: Path,
    slug: str,
    *,
    agent_template: Path | None,
    platforms: list[str] | None = None,
) -> int | None:
    ...
    selected = _prompt_platforms() if platforms is None else platforms
    _install_platform_instructions(cwd, slug, selected)
    ...
```

Teammate-clone path that only installs hooks (manifest present) stays unchanged — it never calls `_setup_assets` / `_prompt_platforms`.

In `cli.py` init parser:

```python
init_p.add_argument(
    "--platforms",
    help="Comma-separated platform keys (skip interactive prompt)",
)
```

In `_DISPATCH`:

```python
("init", None): lambda a: _load("init", "cmd_init")(
    kb_url=getattr(a, "kb_url", None),
    local=getattr(a, "local", False),
    create_remote=getattr(a, "create_remote", False),
    kb_name=getattr(a, "kb_name", None),
    slug=getattr(a, "slug", None),
    platforms_raw=getattr(a, "platforms", None),
),
```

- [ ] **Step 4: Run tests to verify they pass**

Ask the user to run:

```bash
uv run pytest tests/test_init_platforms.py tests/test_prompt_platforms.py -v
uv run ruff check src/reinicorn/commands/init.py src/reinicorn/cli.py tests/test_prompt_platforms.py tests/test_init_platforms.py
```

Expected: PASS / clean.

- [ ] **Step 5: Commit**

```bash
git add src/reinicorn/cli.py src/reinicorn/commands/init.py tests/test_init_platforms.py
git commit -m "$(cat <<'EOF'
feat(init): add --platforms to skip interactive prompt

EOF
)"
```

## Dependencies

- Spec `honest-init-platform-prompt` must remain **approved** (already true on kb main).
- No dependency on other active exec-plans beyond awareness of
  `feat-worktree-aware-kb-resolution` (unrelated overlap: none expected —
  this branch only touches init prompt/CLI).

## Self-review checklist (author)

- Spec §1–2 UX → Task 1
- Spec §3 parse + warnings → Task 1
- Spec §4 test matrix → Task 1 (`test_prompt_platforms.py`)
- Spec §5 `--platforms` → Task 2
- Non-goals respected (no TUI, no `--harnesses`, no kb-source rewrite)
