---
type: spec
title: 'Harness engineering quick wins: principle provenance, surprises log, feedback triage'
slug: harness-engineering-quick-wins-principle-provenance-surprise
lifecycle: active
status: draft
created: 2026-07-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Harness engineering quick wins: principle provenance, surprises log, feedback triage

## Problem

Research into the post-publication evolution of OpenAI's "Harness Engineering" article (Feb 2026) surfaced three guidelines with strong practitioner consensus that reins does not yet encode:

1. **The ratchet principle** (Hashimoto, Osmani, Guo, Kim — the most-repeated addition to the discipline): every agent-facing rule must trace to a specific *observed* failure; speculative rules waste context and rot. Reins' golden principles have a "Why" per principle but no failure provenance and no pruning discipline. Reins also has no stated position on the open shrink-vs-migrate debate (Anthropic: strip harness components as models improve; Osmani: harnesses migrate rather than shrink).
2. **Surprises and discoveries as first-class plan artifacts** (OpenAI Codex cookbook PLANS.md spec): reins plan templates capture decisions (`decisions.md`) but have no home for mid-execution surprises — exactly the field the kb-aggregation bet needs most, since surprises are where the workflow's blind spots reveal themselves.
3. **Structured license to push back on review feedback** (Lopopolo's own post-article practice): the `receiving-code-review` skill permits pushback in prose but gives agents no severity vocabulary, so feedback triage is ad hoc and scope creep from reviewers has no principled off-ramp.

## Design Goals

- Every golden principle carries a stated provenance (the observed failure that motivated it), enforced by template shape and a lint warning; existing principles are honestly labeled as founding principles.
- Principles that stop earning their context cost have a defined retirement path that preserves the historical record.
- Every execution plan has a durable, structured place to record surprises discovered during implementation, harvestable by retros and future kb aggregation.
- Agents receiving code review triage feedback by severity and have explicit license to defer or decline low-severity items, with deferred items captured in the kb rather than dropped.
- All three changes flow to new installs via the seed templates, not just this repo's kb.

## Design

### 1. Ratchet provenance for golden principles

**`kb/reins/golden-principles.md` (this repo) and the seed text in `src/reins/kb_seed.py`:**

- Intro gains the ratchet rule: a new principle must cite the observed agent failure that motivated it. If no failure has been observed, it is not yet a principle (capture it as an idea instead).
- Every principle entry gains a `**Provenance:**` line. The seven existing principles get `Provenance: founding principle (OpenAI harness engineering, Feb 2026)`.
- New `## Retired Principles` section at the bottom, with guidance: periodically re-test whether a principle's failure mode still reproduces with current models; a principle that no longer reproduces (or whose enforcement cost exceeds its value) moves here with a dated retirement note instead of being deleted. This is reins' explicit position on shrink-vs-migrate: **accrete by ratchet, prune by evidence.**

**`_create_principle` in `src/reins/commands/doc_create.py`:**

- Appended entry template gains a `Provenance:` placeholder line alongside the existing `Prevents:` line, so `reins principle add` output prompts for it structurally. No interactive prompting; the placeholder plus lint rule below carry the enforcement.

**New lint rule (`src/reins/linter/rules/`):**

- Warns (not errors) on any principle entry in `golden-principles.md` lacking a `Provenance:` line. Must handle both entry formats in the wild: `### N. Title` headings (curated file) and `N. **Title**` list items (appended by `reins principle add`). Follows the existing rule pattern in `plan_structure.py` / `docs_freshness.py`.

### 2. Surprises & Discoveries in plan templates

- `kb/reins/exec-plans/_template/progress.md` (this repo) and the seed `progress.md` in `kb_seed.py` gain:

  ```markdown
  ## Surprises & Discoveries

  <!-- Things learned during execution that the plan didn't anticipate. -->
  - [date] — [what surprised us] → [implication or action taken]
  ```

- Home is `progress.md` because it is the living during-execution doc: retro's "Lessons Learned" is post-hoc, and `decisions.md` records choices, not discoveries.
- The retro template's "Lessons Learned" section gains a one-line pointer: harvest from `progress.md` → Surprises & Discoveries.
- Not added to `REGISTRY["plan"].required_sections` (that governs `plan.md` lint checks, not `progress.md`); no lint enforcement for this section.

### 3. P0/P1/P2 feedback triage in receiving-code-review

`.claude/skills/receiving-code-review/SKILL.md` gains a "Triage by Severity" subsection after "The Response Pattern":

- **P0 — correctness, security, data loss:** fix now, in this PR, no debate.
- **P1 — legitimate improvement in scope:** fix in this PR.
- **P2 — preference, polish, or scope creep:** explicit license to defer or decline with technical reasoning. Deferred P2s are captured via `reins idea create` or `reins debt create` so they survive the PR.
- Existing verification-before-implementing flow is unchanged; triage happens after VERIFY/EVALUATE, informing RESPOND/IMPLEMENT.
- The skill keeps its `upstream: superpowers/v5.0.7` marker; the divergence is a reins-specific convention for `update-superpowers` to preserve on future syncs.

## Non-Goals

- No doc-gardening agent, SHA-stamping, or drift detection (separate spec; ideas already captured).
- No spec-granularity guidelines (separate spec; idea captured).
- No evaluator-in-fresh-context rework of `verification-before-completion` (candidate follow-up).
- No interactive provenance prompting or hard lint errors — the ratchet is enforced by template shape plus a lint warning, not by blocking workflows.
- No retroactive rewriting of existing kb plan docs; new sections apply to templates and future plans only.
