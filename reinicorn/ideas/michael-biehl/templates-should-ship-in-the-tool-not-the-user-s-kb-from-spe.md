---
type: idea
title: Templates should ship in the tool, not the user's kb — from SpecKit
slug: templates-should-ship-in-the-tool-not-the-user-s-kb-from-spe
lifecycle: active
status: new
created: 2026-07-30
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Templates should ship in the tool, not the user's kb — from SpecKit
## Description

Reinicorn seeds doc templates into the user's kb at `rcorn init` (via
`kb_seed.py`). After that they are the user's copies. When a template improves
upstream — a better exec-plan shape, a new frontmatter field, a corrected
review-status block — the improvement does not reach anyone who already ran
`init`. Every existing kb silently keeps the old shape.

SpecKit keeps `.specify/templates/` inside the install rather than the
project, with a layered override system: project-local overrides > presets >
extensions > core defaults. Template fixes arrive through `specify self
upgrade` like any other code change, and a project only carries the deltas it
deliberately chose.

## Why this matters more than it looks

This is a slow leak that gets worse with every user, and unlike the extension
surface it is not a feature that can simply be added later — changing where
templates live is a migration for every existing kb. The cost of fixing it
rises with adoption. Of everything surfaced in the four-tool comparison (see
[[2026-05-04-feature-mood-board-from-agent-os-bmad-speckit]]), this is the one
worth acting on first.

It also compounds with the linter: `linters/rules/kb` enforces doc shape, but
the templates that produce that shape drift independently of the rules that
check it. Ship both from the tool and they stay in sync by construction.

## Sketch

- Templates live in the reinicorn install (the repo already has `templates/`
  for `AGENTS.md` — extend that to doc templates).
- `kb/_template/` becomes an override layer, not the source: present only when
  a project deliberately customizes a template.
- Resolution order: kb-local override > tool default.
- `rcorn update` picks up template changes with everything else; `rcorn kb
  status` reports which templates a kb has overridden and whether any override
  is based on a stale upstream version.
- Needs a migration path for existing kbs: detect a `_template/` that matches
  a known upstream version and retire it; keep it as an override if it has
  diverged.

## Notes

Surfaced during the 2026-07-30 four-tool comparison. Originally flagged in the
2026-05-04 mood board as a cross-cutting theme and still unaddressed as of
2026-07-30 — re-raised here as a standalone idea because it is the highest-cost
item to defer.
