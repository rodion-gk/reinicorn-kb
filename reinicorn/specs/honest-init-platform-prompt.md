# Honest init platform prompt

**Date:** 2026-07-20
**Author:** Rodion Izotov
**Status:** approved
**Origin:** ai-assisted
**Human-validated:** true 

## Problem

`rcorn init` asks which AI coding platforms to install instructions for via
`_prompt_platforms()` in `src/reinicorn/commands/init.py`. The prompt prints
checkbox-looking markers (`[x]` / `[ ]`) next to a numbered list, then waits
on plain `input()` for comma-separated numbers (or Enter to keep defaults).

That visual affordance implies an arrow-key / space-to-toggle TUI. Users who
try keyboard navigation discover the prompt is actually typed-input only.
The adjacent kb-source prompt (`Choose [1/2/3]:`) is already honest about
typed selection; the platform prompt is not.

The current parse model is also a **toggle** (typed numbers flip defaults
on/off). The only user-visible cue is the word "Toggle", which is easy to
miss; typing `2` is naturally read as "I want Cursor", not "add Cursor while
keeping Claude". Select-set semantics match that intuition better.

Reinicorn has zero runtime dependencies, so a real interactive multi-select
library is out of scope for this fix.

`rcorn init` already has non-interactive escapes for the kb-source step
(`--kb-url`, `--local`, `--create-remote`), but `_setup_assets` always
calls `_prompt_platforms()`. Scripts, CI, and agents still cannot complete
init without a TTY answer for platforms.

## Design Goals

- The platform prompt must not look like an interactive checkbox TUI.
- Typed numbers mean **select-set**: the listed numbers are the chosen
  platforms (replacing defaults), not toggles. Empty Enter keeps the
  defaults.
- Prompt copy must state that model in plain language (no "toggle").
- Defaults remain visible in the prompt line (today: Claude Code only).
- `rcorn init --platforms <keys>` skips the interactive platform step
  (same role `--kb-url` plays for kb source).
- Zero new runtime dependencies.
- Once a selection is chosen, install behavior for those platforms is
  unchanged.

## Design

### 1. Plain numbered list

Replace checkbox markers with a plain numbered list:

```
Which AI coding platforms do you use?

  1) Claude Code
  2) Cursor
  3) GitHub Copilot
  4) Codex

Enter numbers to select (e.g. 1,2), or Enter for default [Claude Code]:
```

Option keys, labels, and default flags stay as they are today
(`claude` default-on; `cursor`, `copilot`, `codex` default-off).

### 2. Prompt line carries defaults and interaction model

The input line states:

1. That typed numbers are the selection (comma-separated allowed).
2. That bare Enter keeps the default.
3. Which platforms are default-selected, in brackets using human labels
   (e.g. `[Claude Code]`). If multiple defaults ever exist, join labels
   with commas inside the brackets.

No per-row selected/unselected decoration in the list. Do not use the word
"toggle".

### 3. Select-set parse logic

Replace toggle semantics with select-set.

**Parse order (pinned — do not fuse whitespace):**

1. Strip leading/trailing whitespace from the whole input.
2. If empty → return the default-selected platform keys (no warning —
   intentional confirm-defaults path).
3. Else **split on commas first**, then **strip each token**. Do **not**
   remove spaces before splitting (today’s
   `raw.replace(" ", "").split(",")` is rejected: it silently turns
   `2 3` into token `23`).
4. For each non-empty token: if it is all digits and the 1-based index is
   in range, select that option; otherwise discard it.
5. Deduplicate selected options; return keys in option-list order (not
   input order).

Discarded tokens are **not silent**: print a warning via `console.warn`
that names the discarded token(s). If the input was non-empty but yielded
no valid indices, print that warning and fall back to defaults (still no
re-prompt loop).

Warning copy must say what was discarded and what was kept, e.g.:

- `abc` → warn that the input was ignored and defaults are used
  (`Claude Code`).
- `1,abc,9` → warn that `abc` and `9` were ignored; selection is still
  `["claude"]` from the valid token.
- `2 3` → one token `"2 3"` (not `"23"`); non-digit → warn + defaults.
- `2, 3` → tokens `"2"` and `"3"` → `["cursor", "copilot"]`, no warning.

Use the existing `reinicorn.console.warn` helper (stderr progress/debug
surface is fine; the point is a human/agent-visible line, not silence).

Examples with today’s defaults:

| Input | Result | Warning? |
|-------|--------|----------|
| `` (Enter) | `["claude"]` | no |
| `2` | `["cursor"]` | no |
| `1,2` | `["claude", "cursor"]` | no |
| `2,4` | `["cursor", "codex"]` | no |
| `2, 3` | `["cursor", "copilot"]` | no |
| `2 3` | `["claude"]` (defaults) | yes — token `2 3` discarded |
| `abc` | `["claude"]` (defaults) | yes — input discarded |
| `1,abc,9` | `["claude"]` | yes — `abc`, `9` discarded |

### 4. Tests

Add a focused unit test for `_prompt_platforms` that mocks `input` and
captures stdout/stderr, asserting:

- Output does not contain `[x]` or `[ ]` checkbox markers.
- Output prompt uses select wording (e.g. contains `select` / `default`,
  does not contain `toggle` case-insensitively). These wording checks
  intentionally couple the test to honest prompt copy — they will break
  on future rewording; that is acceptable because misleading copy is the
  bug this spec fixes. Prefer stable substrings over full-string equality.
- Empty input returns the default selection (`["claude"]` with today’s
  defaults) and emits no discard warning.
- Input `"2"` returns `["cursor"]` only (select-set, not toggle).
- Input `"1,2"` returns `["claude", "cursor"]`.
- Input `"9"` returns `["claude"]` with a discard warning (out-of-range;
  non-empty input, no valid indices → defaults).
- Input `"2,2"` returns `["cursor"]` (dedup).
- Input `"2,1"` returns `["claude", "cursor"]` (option-list order, not
  input order).
- Input `"abc"` returns `["claude"]` and emits a warning that the input
  was discarded / defaults apply.
- Input `"1,abc"` returns `["claude"]` and emits a warning mentioning
  the discarded token.
- Input `"2, 3"` returns `["cursor", "copilot"]` with no warning
  (split-then-strip-per-token).
- Input `"2 3"` returns `["claude"]` with a warning (single non-digit
  token; must **not** be parsed as `23`).

Existing init tests that mock `_prompt_platforms` remain unchanged.

### 5. `--platforms` flag (non-interactive escape)

Add `rcorn init --platforms <keys>` so init can skip `_prompt_platforms`
entirely. Naming is **`--platforms`**, not `--harnesses`: the code and
assets already use "platform" (`_prompt_platforms`, `PLATFORM_FILES`,
`PLATFORM_TEMPLATES`, `platform-instructions/`).

**CLI**

- Flag: `--platforms KEYS` on the existing `init` parser in
  `src/reinicorn/cli.py`.
- `KEYS` is a comma-separated list of platform keys using the **same**
  split-then-strip-per-token rule as §3 (e.g. `claude, cursor` is fine;
  do not strip spaces before splitting).
- Omitted → interactive `_prompt_platforms()` as today (with the new UX).
- Present → do not call `_prompt_platforms()`; use the parsed key list.

**Validation (boundary)**

Validate at the CLI / `cmd_init` boundary before asset setup:

- Each key must be one of the known option keys:
  `claude`, `cursor`, `copilot`, `codex`.
- Unknown keys → hard error via `console.error` (what / which key / known
  keys), return non-zero. Do **not** warn-and-continue like the
  interactive discard path — a bad flag is a caller bug.
- Duplicates are deduped; order follows the canonical option-list order,
  not the flag order.
- Empty value (`--platforms ''` or `--platforms` with empty string) means
  "install no platform instruction files" (`[]`) — valid, matches today’s
  empty selection.

**Plumbing**

- Thread `platforms: list[str] | None` through `cmd_init` →
  `_setup_assets`. When `None`, prompt; when a list (including empty),
  install that list.
- Teammate-clone path that only installs hooks (manifest already present)
  is unchanged — it never prompts for platforms today.

**Tests**

- `--platforms cursor` skips the prompt (mock `input` unused / not called)
  and installs Cursor instructions.
- `--platforms nope` fails with a non-zero exit and an error naming the
  unknown key.
- Omitting the flag still reaches `_prompt_platforms` (covered by existing
  mocks / the unit tests above).

## Non-Goals

- No interactive TUI (arrow keys, space toggle, cursor libraries).
- No new runtime dependencies (`questionary`, `rich`, etc.).
- No `--harnesses` alias (use `--platforms`).
- No rewrite of other init prompts (`_prompt_kb_source`, gh auth, etc.),
  except that this prompt should remain stylistically consistent with them.
- No change to which platform instruction files are generated once a
  selection is chosen.
- No re-prompt loop on invalid interactive input — warn and continue
  (defaults or partial valid set), do not ask again.
