---
type: idea
title: Document provenance tracking for all harness docs. Every doc needs frontmatter t
slug: 2026-02-21-document-provenance-tracking-for-all-harness-docs-every-doc
lifecycle: active
status: new
created: 2026-02-21
author: Michael Biehl
---

# Document provenance tracking for all harness docs. Every doc needs frontmatter t

## Description

Document provenance tracking for all harness docs. Every doc needs frontmatter that clearly distinguishes: author (human name or 'ai-generated'), origin ('human-written' | 'ai-drafted' | 'ai-assisted' | 'ai-generated'), human-validated (true/false + validator name + date). Skills and CLAUDE.md rules must enforce: agents always mark their output as ai-generated/ai-drafted, never self-validate, always prompt for human validation. Lint rule to flag docs missing provenance fields. This is about trust signals — not gatekeeping. Unvalidated docs can still be merged and used, but the provenance metadata lets agents and humans weight content appropriately when parsing and synthesizing later. A validated human-reviewed doc carries more authority than a raw AI draft.

## Template-driven doc creation

All harness docs must be created through `reins` commands (e.g. `reins plan create`, `reins idea`, etc.) — **agents should never create harness files directly** using Write/Edit tools. Each command scaffolds the doc from a template that includes the correct frontmatter, directory placement, and structure. This ensures:

- Provenance fields are always present from the start (no retroactive lint fixes)
- Consistent structure across all doc types
- The reins CLI is the single chokepoint for doc creation, making it easy to enforce rules

If a doc type doesn't have a `reins` command yet, the right fix is to add one — not to bypass the convention.

## Repo/branch scoped directories

All working docs (exec-plans, progress, decisions, touched-areas) must be placed in **repo/branch scoped directories**, not flat or global locations. This was part of the original design (see `exec-plans/active/{branch-name}/` in reins-design.md) and must extend to all doc types that are branch-specific:

```
harness/reins/exec-plans/active/{repo}/{branch-name}/
    plan.md
    progress.md
    decisions.md
    touched-areas.md
```

The `{repo}` scope is needed because a single harness submodule can serve multiple consuming repos. Without repo scoping, branches from different repos with the same name would collide. The reins commands handle directory creation and scoping automatically — another reason agents must use the commands rather than creating files directly.

## Notes

- The repo scope was part of the original multi-repo design but was lost during MVP simplification. This idea re-establishes it as a requirement.
- `reins plan create` should auto-detect repo name and branch, create the scoped directory, and scaffold from template.
- Lint rule: flag any harness doc that exists outside its expected scoped directory.
