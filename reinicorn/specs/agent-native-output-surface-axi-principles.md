---
type: spec
title: Agent-native output surface (AXI principles)
slug: agent-native-output-surface-axi-principles
lifecycle: done
status: implemented
created: 2026-07-02
author: Michael Biehl
origin: ai-assisted
human_validated: true
---

# Agent-native output surface (AXI principles)

## Problem

The primary consumer of the reins CLI is a coding agent, but its output surface is
designed for humans. Concretely:

- `reins` with no arguments prints an argparse usage manual, not state. An agent
  running `reins` to orient itself learns the menu, not the situation, and burns a
  follow-up call.
- The SessionStart hook (`.claude/hooks/session-start.sh`) syncs the kb but injects
  nothing into agent context. Sessions start blind; orientation depends on skills
  being loaded and followed.
- Next-step hints exist ad hoc (`kb status` suggests `reins plan create`) but are not
  a convention. Most commands end without telling the agent what it can do next.
- Errors go to stderr via `console.error`; axi puts structured errors on stdout,
  the channel the agent treats as data. No-op audit is missing: re-running
  `hooks install`, `mode enable`, etc. should be explicit exit-0 no-ops. Error
  messages are not consistently structured (what/where/how-to-fix) and can leak
  raw git/dependency output.
- There is no way to read a kb doc through the CLI. Agents fall back to raw file
  reads, which bypasses any chance to truncate, cross-link, or guide next steps.

This mirrors the gap analysis against the AXI spec
(https://github.com/kunchenguid/axi): principles 3 (truncation), 6 (structured
errors/idempotency), 7 (ambient context), 8 (content first), and 9 (contextual
disclosure) are unmet. Principles 1–2 (TOON, minimal schemas) are adapted, not
adopted — reins outputs are small; the token win is in eliminating follow-up calls,
not compressing rows.

## Design Goals

- An agent that runs bare `reins` sees live state (branch plan status, kb health,
  overlap warnings) plus 2–3 parameterized next steps — never a usage manual.
- Every Claude Code session in a reins repo starts with a compact kb dashboard
  already in context, without any skill needing to fire.
- Every list/mutation command ends with a `next:` footer of relevant, complete,
  parameterized commands. Detail views and confirmations that are self-contained
  omit it.
- Agents can read any kb doc via the CLI with a truncated preview by default and a
  `--full` escape hatch, with truncation explicitly signaled (`… (truncated, N chars
  total)`).
- Axi channel model: all agent-consumed output (data, errors, suggestions) on
  stdout; stderr carries only progress/debug diagnostics. Idempotent mutations
  no-op with exit 0 and say so. Exit codes: 0 success (incl. no-ops), 1 error,
  2 usage error.
- Output degrades to a stable, compact, unstyled form when stdout is not a TTY
  (colors already gate on isatty; decorative indentation/headers should too).

## Design

### 1. `show` verbs (new)

Per-type verbs, consistent with the noun-first structure and driven by the
`doc_types.py` registry — no top-level `reins show`:

- `reins spec show <slug>` / `reins prd show <slug>` / `reins debt show <slug>`
- `reins plan show [branch]` (defaults to current branch)
- `reins retro show [branch]`, `reins idea show <slug>`
- Companion `list` verbs where missing: `reins spec list`, etc., emitting minimal
  schemas (slug, title, status, date) with a total count.

Behavior (AXI principle 3):

- Default: front-matter + first ~1500 chars of body, then
  `… (truncated, <N> chars total)` and a hint line: `next: reins spec show <slug> --full`.
- `--full` prints the entire doc.
- Unknown slug: exit 1 with the valid slugs (or nearest matches) inline — the error
  self-corrects instead of pointing at `--help`.
- Implementation: one shared handler in `commands/doc_show.py` keyed off the
  registry (like `_doc_group_with_create_title` does for create).

### 2. Content-first home view

Bare `reins` (no args) prints, in order:

1. `bin:` path and one-line description (AXI principle 10 home-view header).
2. Live state: current branch, this branch's plan status, active plan count,
   overlap warnings if any, lint/stale-doc counts (reuse `kb status` internals).
3. `next:` footer with 2–3 suggestions conditioned on state (no plan → `reins plan
   create`; plan exists → `reins plan status`; outside a reins repo → `reins init`).

Constraint: the home view is local-reads-only — no fetch, no network, no slow git
operations. Bare `reins` must stay near-instant or it stops being the orientation
command for both agents and humans.

`reins help` and `reins --help` keep the argparse manual. Outside a reins-enabled
repo, bare `reins` states that definitively and suggests `reins init`.

### 3. Ambient session context

Add `reins kb status --compact`: a ≤10-line, no-decoration dashboard (branch, plan
state, overlaps, lint failures, stale count). Extend `session-start.sh` to print it
to stdout after the kb sync, so Claude Code injects it as session context. Budget
rule: this loads every session — keep it ruthlessly minimal; deep data stays behind
explicit commands. Other platforms get the same via their hook mechanisms when their
platform-instructions are installed (do not hard-code Claude-only).

### 4. Contextual disclosure convention

House rule: list and mutation outputs end with a `next:` block of 1–3 complete,
parameterized commands (`<slug>`, `"<title>"` placeholders; never guessed values).
Self-contained outputs (detail views, counts, pure confirmations) omit it. Suggest
variety, not a fixed workflow. Applied across existing commands (`create` →
follow-on doc/plan suggestions; `sync` conflict → the exact resolving command).

### 5. Error and exit-code audit

- Route errors to stdout in the same structured shape as data (`error: …` +
  `next: …` fix suggestion), stated as what/where/how-to-fix. Stderr is reserved
  for progress/debug diagnostics only; failure status is carried by exit codes,
  not by channel. `console.py` gains an explicit progress/debug channel to make
  the split mechanical.
- TTY detection may affect decoration only (color, indentation) — never content,
  truncation behavior, or output channel. Agent-observed behavior must be exactly
  reproducible by a human running the same command.
- Audit all mutations for idempotency: `hooks install`, `mode enable|disable`,
  `init` re-run, `update` at current version → explicit `(no-op)` message, exit 0.
- Errors state what/where/how-to-fix (existing golden principle) and never leak
  raw git/dependency stderr; translate to reins commands.

### 6. Enforcement (follow-on)

Once implemented, promote the conventions to mechanically enforced rules per the
"mechanical enforcement over documentation" philosophy: a structural test that every
registered command's output path can emit a `next:` footer, and a test that error
paths emit the structured `error:` shape on stdout with a fix suggestion (stderr
reserved for progress/debug). Add a golden principle only when the lint exists.

## Non-Goals

- TOON or any new serialization format. Outputs stay line-oriented `key: value`;
  the compact win comes from fewer follow-up calls, not row compression.
- JSON output mode (`--json`). Out of scope until an agent consumer needs it.
- MCP server or any non-CLI interface.
- Pagination. Kb collections are small; full listings with a total count suffice.
- Rewriting skills. Skills keep workflow knowledge; this spec only makes the CLI
  self-guiding enough that an agent without skills can still operate safely.
