---
type: spec
title: 'Design: Reins Init Rework'
slug: reins-init-rework
lifecycle: done
status: implemented
created: 2026-02-21
author: Michael Biehl
---

# Design: Reins Init Rework

**Predecessor:** mvp-unified-attach.md
**Trigger:** QA findings from MVP Unified Attach (PR #4)

## Problem

`reins attach` works but the UX is rough:

1. **Naming:** "attach" is unclear. Users think "init" when setting up a project.
   Rename `attach` → `init` (keep `attach` as hidden alias for backward compat).

2. **Interactive prompts are clunky.** `Choose [1/2]:` is a bad TUI pattern.
   Needs proper terminal UX — arrow-key selection, colored options, clear
   descriptions of what each choice does.

3. **Terminal-only.** The command only works interactively in a user's terminal.
   An agentic walkthrough skill (e.g. `/init-project`) should also exist so an
   AI agent can drive the setup flow programmatically via flags/args.

4. **No remote repo creation.** Mode 2 creates a local bare repo but doesn't
   create a GitHub remote. Users have to do that manually later. The command
   should offer to create a private `{name}-harness` repo via `gh repo create`
   and prompt for a custom name.

5. **git protocol.file.allow.** Modern git (2.38.1+) blocks `file://` transport
   in sub-processes. Local bare repo mode needs `-c protocol.file.allow=always`
   on clone, push, and submodule add calls. **Fixed in this QA session.**

## Design Goals

### CLI UX

- **Default mode (no flags):** Rich interactive TUI with arrow-key selection,
  colored output, progress indicators. This is the "human in terminal" path.

- **Flag-driven mode (`--harness-url`, `--local`, `--create-remote`):** Fully
  non-interactive. Every interactive prompt has a flag equivalent. This is the
  "script/agent" path.

- **Agentic skill:** A `/init-project` skill that walks the agent through the
  setup, calling `reins init` with appropriate flags.

### Harness Repo Creation Flow

When user picks "create new harness repo":

1. Ask for repo name (default: `{project}-harness`)
2. Ask visibility (default: private)
3. If `gh` is available: `gh repo create` → push seed content → add as submodule
4. If `gh` unavailable: create local bare repo, print instructions for manual
   GitHub creation later

### Rename

- `reins init` — primary command (new projects AND existing repos)
- `reins attach` — hidden alias, prints deprecation notice
- The old `reins init` (scaffold from scratch) becomes `reins new` or
  is folded into `init` with a `--from-scratch` flag

## TUI Library

Options for better interactive prompts:

- **[simple-term-menu](https://github.com/IngoMeyer441/simple-term-menu)** —
  zero-dep arrow-key menu selection
- **[questionary](https://github.com/tmbo/questionary)** — prompts with
  validation, selection, confirmation
- **[rich](https://github.com/Textualize/rich)** — already considering for
  console output; has `Prompt` and `Confirm`
- **Roll our own** — minimal `curses`-based selector, no deps

Recommendation: start with `questionary` or `simple-term-menu` for selection
prompts. Keep dependency count low — one library max.

## Non-Goals (for this rework)

- Multi-harness support (one repo, multiple harnesses)
- Harness templates / flavors
- GUI / web setup flow
