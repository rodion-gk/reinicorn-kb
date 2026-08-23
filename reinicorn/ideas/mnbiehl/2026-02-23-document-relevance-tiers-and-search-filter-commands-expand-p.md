---
type: idea
title: Document relevance tiers and search/filter commands
slug: 2026-02-23-document-relevance-tiers-and-search-filter-commands-expand-p
lifecycle: active
status: new
created: 2026-02-23
author: mnbiehl
---

# Document relevance tiers and search/filter commands

## Problem

The provenance idea (2026-02-21) gives us authorship trust signals (`author`, `origin`, `human-validated`) but not **document relevance**. An agent doing research today has no way to distinguish an authoritative golden principle from a raw brainstorming idea or a branch-scoped exec plan. One-off plans and draft-level docs get treated with the same weight as canonical decisions, polluting research synthesis.

## Proposed solution

### 1. Relevance-tier frontmatter field

Add a `tier` field to the provenance frontmatter spec:

| Tier | Meaning | Examples |
|------|---------|----------|
| `canonical` | Source of truth, always include in research | AGENTS.md, golden-principles.md, ARCHITECTURE.md |
| `decision` | Approved decisions/specs, include unless stale | Approved design docs, product specs |
| `working` | Active work-in-progress, include only when directly relevant | Draft design docs, active exec-plans |
| `ephemeral` | Brainstorming/scratch, exclude from research by default | Ideas, one-off plans, branch-scoped progress notes |

Default tier assignment by doc type:
- `harness/*.md` root docs → `canonical`
- `design-docs/` with status `approved` → `decision`
- `design-docs/` with status `draft` → `working`
- `ideas/` → `ephemeral`
- `exec-plans/active/` → `working`
- `exec-plans/completed/` → `decision` (they capture what was actually done)

### 2. Reins CLI search/filter commands

```
reins docs list                              # all harness docs with tier + status
reins docs list --tier canonical,decision    # only authoritative docs
reins docs list --validated-only             # only human-validated
reins docs search "submodule"                # full-text search with tier filtering
reins docs search "submodule" --tier decision,canonical  # research-grade results only
```

### 3. Agent research instructions

Skills and CLAUDE.md rules should instruct agents:
- **Research synthesis**: use `reins docs search --tier canonical,decision` — never raw grep across all harness docs
- **Context gathering**: may include `working` tier for the current branch's exec-plan
- **Never cite `ephemeral` tier docs** as evidence for decisions or conventions

## Relationship to existing ideas

- **Extends** provenance tracking (2026-02-21) — adds the missing relevance dimension
- **Depends on** template-driven creation (2026-02-22) — templates set the default tier
- **Feeds into** M5 (MVP roadmap) — trust signals milestone

## Notes

- The tier could be auto-derived from doc type + status in many cases, reducing manual annotation burden
- Design-docs already have a status lifecycle in their index — tier can key off that
- Consider a `reins docs promote` command that bumps tier (e.g. idea → design doc draft → approved)
