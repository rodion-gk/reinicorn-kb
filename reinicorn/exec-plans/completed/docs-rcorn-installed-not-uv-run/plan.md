---
type: plan
title: 'Execution Plan: docs/rcorn-installed-not-uv-run'
slug: docs-rcorn-installed-not-uv-run
lifecycle: done
status: complete
created: 2026-08-15
author: Michael Biehl
branch: docs/rcorn-installed-not-uv-run
ticket: N/A
spec: N/A
---

# Execution Plan: docs/rcorn-installed-not-uv-run

## Goal

Make "use the installed `rcorn`, not `uv run rcorn`" an explicit repo rule in
AGENTS.md, so agents and contributors don't mask a stale installed binary.

## Acceptance Criteria

- [x] AGENTS.md states the rule: installed `rcorn` always, `uv run rcorn` only
      when explicitly testing in-repo changes before installation.
- [x] The rule explains why (hooks and daily usage call the global binary) and
      the remedy (`uv tool install --force .` after merging CLI changes).

## Approach

One-line policy existed as personal agent memory only; after merging PR #48 the
installed tool sat one version behind while all session commands ran through
`uv run`, and the git hooks kept calling the stale global binary. Promote the
rule to AGENTS.md where every agent loads it.

## Tasks

- [x] Rewrite the AGENTS.md "Run the CLI" bullet with the rule, rationale, and
      reinstall step.
- [ ] PR and merge.

## Dependencies

None — docs-only; follows up the PR #48 kb-clone migration.
