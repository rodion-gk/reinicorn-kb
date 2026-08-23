---
type: plan
title: 'Execution Plan: docs-readme-doc-governance'
slug: docs-readme-doc-governance
lifecycle: active
status: in-progress
created: 2026-08-20
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: docs-readme-doc-governance
ticket: N/A
spec: reinicorn/specs/skill-base-agnostic-reinicorn-adapter-infrastructure-for-ext.md
---

# Execution Plan: docs-readme-doc-governance

## Goal
Bring README.md in line with the skill-agnostic Reinicorn that landed in #57:
drop the "SDD core" framing (the forked superpowers skills are gone; methodology
now comes from a pluggable adapter) and re-center the pitch on doc governance —
the kb, the registry-driven doc types, mechanical enforcement, and the review lane.

## Acceptance Criteria
- [x] README no longer describes a spec-driven-development workflow as Reinicorn's core
- [x] README leads with doc governance: where docs live, how they are created, how they are enforced and reviewed
- [x] Skill-set adapters are described as the (optional) methodology layer, not the product
- [x] Every command and path named in the README exists in the current CLI/repo
- [x] Repository-structure block matches the tree on main

## Approach
Rewrite the intro, "The workflow", and "The skill set" sections; keep the
Quick Start, CLI table, and kb-as-shared-clone sections with only factual
touch-ups. Frame the three things Reinicorn owns (per `using-reinicorn`):
where docs live, how docs are created, which skills gate which docs.

## Tasks
- [x] Rewrite intro around doc governance
- [x] Replace "The workflow" with a governance-oriented section (doc types, enforcement, review lane)
- [x] Rework "The skill set" as the optional methodology layer wired through the registry
- [x] Verify CLI table and repo tree against `rcorn help` and `ls`
- [x] Open PR

## Dependencies
None — docs only. AGENTS.md line 3 has the same stale "spec-driven development" wording; out of scope here, flagged for a follow-up.
