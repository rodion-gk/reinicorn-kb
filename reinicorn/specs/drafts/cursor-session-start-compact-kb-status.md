# Cursor session-start compact kb status

**Date:** 2026-07-28
**Author:** Rodion Izotov
**Status:** in-review
**Origin:** ai-assisted
**Human-validated:** false
**Promotes:** kb/reinicorn/ideas/michael-biehl/2026-07-02-wire-kb-status-compact-into-non-claude-session-hooks-cursor.md
**Review-PR:** https://github.com/crystldm/reinicorn-kb/pull/11

## Problem

Claude Code sessions in a Reinicorn repo start with a compact KB dashboard already
in agent context: `.claude/hooks/session-start.sh` runs `rcorn kb status --compact`
on `SessionStart` and injects the stdout (branch, plan state, overlap, `next:`).

Cursor sessions do not. Reinicorn installs Cursor **PreToolUse** guards via
`rcorn hooks install` (doc-template enforcement, block raw kb git), but
`.cursor/hooks.json` has no `sessionStart` entry. Cursor agents therefore depend
on skills being loaded and followed to orient themselves — an extra turn, and
easy to skip.

This is the deferred follow-up from the agent-native output surface (AXI) work:
`rcorn kb status --compact` is platform-neutral and implemented; only Cursor
hook wiring was left out.

## Design Goals

- Every new Cursor Agent/Composer session in a Reinicorn-enabled repo receives
  the same ≤10-line compact dashboard Claude gets, without manual action.
- Hook installation is idempotent and merged into existing `.cursor/hooks.json`
  by `rcorn hooks install` — same path as PreToolUse hooks.
- Session-start script lives at a neutral path (`.reinicorn/hooks/`), not under
  `.cursor/`, consistent with multi-editor hook support.
- Session start stays **local-reads-only and near-instant**: no kb fetch, no lint
  suite, no stale-doc scan (same budget as `_compact_status()` and AXI decisions).
- Script fails open: missing `uv`, non-Reinicorn repo, or CLI error → exit 0,
  no context injected, session proceeds normally.
- Structural tests lock the source repo's `.cursor/hooks.json`, tracked hook
  copies, and the install path so regressions are caught in CI.
- Upgrading Reinicorn re-applies editor hook config automatically so users receive
  new `sessionStart` entries without a manual step.

## Design

### Compact dashboard (unchanged)

Reuse the existing command and output contract from
`src/reinicorn/commands/status.py::_compact_status()`:

```
reinicorn: branch main — no plan
plans: 2 active in this repo scope
overlap: none
next: rcorn plan create
```

Constraints inherited from AXI (do not expand in this spec):

- ≤10 lines, no headers/decoration
- No network, no per-doc `git log` stale scan
- Stdout is agent-facing data

### Session-start hook script

Add `editor-hooks/session-start-status.sh` — a minimal script whose sole job is
injecting compact status into Cursor session context:

```bash
#!/bin/bash
# Inject rcorn kb status --compact into Cursor sessionStart context.
set -euo pipefail

command -v uv >/dev/null 2>&1 || exit 0

context="$(uv run rcorn kb status --compact 2>/dev/null || true)"
[ -z "$context" ] && exit 0

# Cursor sessionStart expects JSON with additional_context (not raw stdout).
if command -v jq >/dev/null 2>&1; then
  jq -n --arg ctx "$context" '{additional_context: $ctx}'
else
  # jq is expected in dev environments; fail open if absent.
  exit 0
fi
```

**Why a separate script from Claude's `session-start.sh`:** Claude's hook also
kb-pulls, handles remote credential rewrite, runs `uv sync`, and calls
`rcorn hooks install`. Those belong to Claude Code's environment bootstrap, not
to ambient context injection. Cursor gets only the compact dashboard — matching
the AXI latency budget and avoiding side effects on every new Composer session.

**Why JSON output:** Cursor's `sessionStart` hook consumes structured stdout
(`{"additional_context": "..."}` per
[Cursor hooks docs](https://cursor.com/docs/hooks)). Claude injects plain stdout;
the formats differ and the script must emit Cursor's shape.

**jq fail-open vs PreToolUse fail-closed:** PreToolUse hooks (`enforce-doc-templates.sh`,
`block-raw-kb-git.sh`) require `jq` to parse stdin and block or allow tool calls —
missing `jq` breaks enforcement. Session-start context injection is **best-effort**:
without `jq`, the script exits 0 silently and the session proceeds without ambient
context. That asymmetry is intentional — orientation is helpful but not safety-critical.

### Cursor hooks.json entry

After `rcorn hooks install`, `.cursor/hooks.json` includes:

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [ ... existing entries ... ],
    "sessionStart": [
      {
        "command": ".reinicorn/hooks/session-start-status.sh"
      }
    ]
  }
}
```

Notes:

- `sessionStart` entries have no `matcher` (unlike `preToolUse`).
- Path is relative to project root, matching existing PreToolUse convention.
- Optional `timeout` is omitted; compact status must complete in well under one
  second on a typical repo.
- **Multiple `sessionStart` hooks may coexist.** Cursor runs each entry in the
  array. Merge logic must **append** new entries and dedupe only by exact
  `command` path — never replace the whole `sessionStart` bucket. A future
  skill-bootstrap hook can register alongside this one.

### `hooks_install.py` changes

Extend `_install_editor_hooks()` in `src/reinicorn/commands/hooks_install.py`:

1. Copy `session-start-status.sh` from `editor-hooks/` to
   `.reinicorn/hooks/` alongside the PreToolUse scripts (same copy/idempotency
   logic, including executable bit).
2. Generalize `_merge_cursor_settings()` to accept an event name parameter
   (default `"preToolUse"` for backward compatibility). Merge `sessionStart`
   entries separately from `preToolUse`.
3. Dedupe `sessionStart` by `command` path only — append if the command is new;
   leave unrelated `sessionStart` entries untouched.

Do **not** register `sessionStart` for Claude or Copilot in this spec — Cursor
only.

### Source repository wiring (dogfooding)

The Reinicorn repo uses the same two-layer hook layout as consumer projects:

```
editor-hooks/session-start-status.sh   ← canonical source (packaged)
.reinicorn/hooks/session-start-status.sh   ← tracked copy, byte-identical
.cursor/hooks.json                       ← points at .reinicorn/hooks/
```

Implementation must update **all three**:

1. Add `editor-hooks/session-start-status.sh`.
2. Add the tracked copy at `.reinicorn/hooks/session-start-status.sh` (must
   match `editor-hooks/` byte-for-byte, same as existing PreToolUse hooks).
3. Add the `sessionStart` entry to `.cursor/hooks.json`.

Extend `tests/test_source_editor_integrations.py`:

- Add `session-start-status.sh` to the hook inventory (extend `HOOK_NAMES` or
  a parallel constant).
- Assert `.cursor/hooks.json` contains `hooks.sessionStart` with command
  `.reinicorn/hooks/session-start-status.sh`.
- Assert `test_source_hooks_match_package_owned_editor_hooks` covers the new
  script (byte match between `editor-hooks/` and `.reinicorn/hooks/`).
- Assert the script calls `uv run rcorn kb status --compact`.

### Shell unit test

Add `tests/hooks/test_session_start_status.sh` (or pytest wrapper invoking bash)
that exercises the script without a live Reinicorn repo:

- **Happy path:** `PATH` includes a stub `uv` that prints known compact output;
  assert stdout is valid JSON with `additional_context` containing that output.
- **No uv:** assert exit 0, empty stdout.
- **Empty context:** stub `uv` that prints nothing; assert exit 0, empty stdout.
- **No jq:** assert exit 0, empty stdout (fail-open).

Run via `tests/run-all.sh` / pytest as appropriate for shell tests in this repo.

### `hooks_install` integration test

Add to `tests/commands/test_hooks_install.py`:

- After `cmd_hooks_install()` in a fresh tmp repo with Cursor config, assert
  `.cursor/hooks.json` contains `hooks.sessionStart` with the expected command.
- Re-run install; assert idempotency (still exactly one matching entry).

### `rcorn update` integration

After a successful asset sync in `cmd_update()` (`src/reinicorn/commands/update.py`),
invoke `cmd_hooks_install()` automatically. Rationale:

- `rcorn update` already syncs `editor-hooks/` → `.reinicorn/hooks/` via the
  manifest, but does not merge entries into `.cursor/hooks.json`.
- `hooks install` is idempotent and fast; running it after update ensures editor
  config picks up new hook registrations (including `sessionStart`) without a
  manual step.
- On no-op update (`Already up to date`), skip the hooks re-run.

Add a test in `tests/commands/test_update.py` (or extend existing update tests):
after update changes hook assets, `.cursor/hooks.json` gains `sessionStart`.

Emit `next: rcorn hooks install` only when update skips the automatic re-run
(e.g. error path) — not when it succeeds and hooks install ran inline.

Document in GETTING-STARTED.md: `rcorn update` re-applies editor hooks after
syncing assets.

### Platform instructions

Append to `platform-instructions/cursor.md` (installed by `rcorn init` as
`.cursor/rules/reinicorn.mdc`):

```markdown
## Session bootstrap

On each new Agent/Composer session, a `sessionStart` hook injects
`rcorn kb status --compact` into context (branch, plan state, overlap, next
step). Installed by `rcorn init` and refreshed by `rcorn update`. Requires
Cursor 2.4+, `uv` on PATH, and `jq` for context injection.
```

Do not change Claude or Copilot platform instructions in this spec.

### Init behavior

No new `rcorn init` flags. Cursor users who selected Cursor during init already
get platform instructions; `rcorn hooks install` (run during init and after
`rcorn update`) picks up the session-start hook automatically.

## Acceptance criteria

- [ ] `editor-hooks/session-start-status.sh` exists; emits valid Cursor
      `sessionStart` JSON when `rcorn kb status --compact` succeeds.
- [ ] `.reinicorn/hooks/session-start-status.sh` tracked copy matches
      `editor-hooks/` byte-for-byte.
- [ ] `rcorn hooks install` copies the script to `.reinicorn/hooks/` and merges
      `sessionStart` into `.cursor/hooks.json` idempotently.
- [ ] `test_hooks_install.py`: fresh repo gets `sessionStart` entry after
      `rcorn hooks install`; re-run is idempotent.
- [ ] Shell test (`tests/hooks/test_session_start_status.sh`): valid JSON on
      happy path; silent exit 0 without `uv`, empty context, or missing `jq`.
- [ ] Reinicorn source repo `.cursor/hooks.json` includes the `sessionStart`
      entry; `test_source_editor_integrations.py` passes.
- [ ] `rcorn update` invokes `hooks install` after a successful asset sync;
      integration test confirms `sessionStart` appears in `.cursor/hooks.json`.
- [ ] `platform-instructions/cursor.md` and GETTING-STARTED.md document session
      bootstrap and the update path.
- [ ] Manual smoke: open a new Cursor Agent session in this repo; compact
      dashboard appears in initial context (verify via agent behavior or hook
      debug output).

## Risks and mitigations

| Risk | Mitigation |
|------|------------|
| Cursor `sessionStart` required v2.4+; older builds reject the hook type | Document minimum version in platform instructions |
| Known Cursor race where `additional_context` may not surface in some builds | Fail open; agents can still run `rcorn kb status --compact` manually; track upstream fix |
| Cursor cloud/background agents may defer or skip `sessionStart` | Fail open; document that local Agent sessions are the primary target |
| `jq` not installed | Fail open (no injection); PreToolUse hooks still require jq for enforcement — document both behaviors |
| Hook runs on every new Composer session — latency | Reuse existing compact path; no network or O(docs) git operations |
| Multiple `sessionStart` hooks — merge clobbers user entries | Dedupe by `command` path only; append, never replace the whole bucket |

## Non-Goals

- **Copilot, Gemini, Codex** session-start wiring — separate follow-up specs.
- **Claude `session-start.sh` changes** — already implemented; leave as-is.
- **`session-start-bootstrap` / `using-reinicorn` skill injection** — different
  concern (skill content vs live repo state); may register as a second
  `sessionStart` entry in a follow-up; this spec's merge logic must not block that.
- **Kb pull/sync on Cursor session start** — too slow and side-effectful; agents
  use `rcorn kb sync` explicitly when needed.
- **Dynamic `next:` from plan checkboxes** — see idea
  `2026-07-04-extend-next-beyond-the-cli-git-hooks-and-dynamic-task-handoff`.
- **`beforeSubmitPrompt` fallback** for pre-2.4 Cursor — not worth the
  complexity; document version requirement instead.
