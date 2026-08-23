---
type: spec
title: 'Design: Skills Fork + Template-Driven Docs'
slug: skills-fork-and-template-docs
lifecycle: done
status: implemented
created: 2026-02-23
author: Michael Biehl + Claude
origin: ai-assisted
human_validated: false
implemented_by: feature-mvp
---

# Design: Skills Fork + Template-Driven Docs

## Problem

Agents currently hand-write harness docs with inconsistent frontmatter, wrong
paths, and no provenance tracking. Meanwhile, reins depends on the external
`superpowers` plugin for core development skills (brainstorming, TDD, debugging,
etc.) which don't reference reins conventions.

Three issues converge:

1. **No doc templates.** Agents guess at frontmatter fields, miss provenance
   metadata, and place files in wrong directories.
2. **External skill dependency.** Superpowers skills reference `docs/plans/`
   instead of `harness/`, don't know about golden principles or `reins`
   CLI commands.
3. **No cross-repo support.** The harness flat structure (`harness/reins/design-docs/`)
   doesn't support multiple repos sharing one harness.

## Design Goals

- Agents **never** hand-write harness docs — all creation goes through
  `reins doc create <type>`.
- Reins ships its **own skill set** (forked from superpowers + custom),
  fully integrated with reins CLI and harness conventions.
- The harness supports **repo-scoped paths** (`harness/{repo}/...`) for
  cross-repo usage from day one.
- An **enforcement hook** blocks agents from creating harness docs directly,
  forcing the CLI workflow.

## Architecture

### Repo-Scoped Harness Paths

Migrate from flat structure to repo-scoped:

```
harness/
├── reins/                     # derived from git remote origin
│   ├── architecture/
│   ├── design-docs/
│   │   ├── index.md
│   │   └── *.md
│   ├── exec-plans/
│   │   ├── active/{branch}/
│   │   └── completed/{branch}/
│   ├── ideas/{username}/
│   ├── product-specs/
│   ├── tech-debt/
│   ├── golden-principles.md
│   └── quality-scores.md
├── {other-repo}/                  # future repos sharing this harness
│   └── ...
└── ideas/index.md                 # shared index (stays at root)
```

**Repo slug derivation:**

```python
url = run_git("remote", "get-url", "origin").stdout.strip()
# git@github.com:mnbiehl/reins.git → "reins"
# https://github.com/mnbiehl/reins.git → "reins"
```

### `reins doc create <type> "title"`

CLI command that generates doc templates with correct provenance, paths, and
structure.

| Type | Output Path | Key Sections |
|------|------------|-------------|
| `design` | `harness/{repo}/design-docs/{slug}.md` | Problem, Design Goals, Design, Non-Goals |
| `plan` | `harness/{repo}/exec-plans/active/{branch}/plan.md` | Acceptance Criteria, Tasks |
| `idea` | `harness/{repo}/ideas/{username}/{date-slug}.md` | Description, Notes |
| `spec` | `harness/{repo}/product-specs/{slug}.md` | User Stories, Acceptance Criteria, Out of Scope |
| `debt` | `harness/{repo}/tech-debt/by-domain/{slug}.md` | Severity, Domain, Impact, Remediation |
| `retro` | `harness/{repo}/exec-plans/completed/{branch}/retro.md` | Outcome, What Worked, What Didn't, Follow-ups |
| `principle` | `harness/{repo}/golden-principles.md` (append) | Rule, Prevents |

**Provenance frontmatter** (auto-filled on every template):

```markdown
**Date:** {today}
**Author:** {git user.name}
**Status:** draft
**Origin:** ai-assisted
**Human-validated:** false
```

After creation: auto-commits to harness, prints file path.

**`reins idea "text"`** becomes a convenience alias for
`reins doc create idea "text"`.

### Agent Enforcement Hook

A `PreToolUse` Claude Code hook on `Write|Edit` that blocks direct creation of
new harness docs:

```bash
#!/usr/bin/env bash
# .claude/hooks/enforce-doc-templates.sh
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_params.file_path // .tool_params.filePath // empty')
[ -z "$FILE" ] && exit 0

case "$FILE" in
  */harness/*/design-docs/*.md|*/harness/*/product-specs/*.md| \
  */harness/*/exec-plans/*/plan.md|*/harness/*/exec-plans/*/retro.md| \
  */harness/*/tech-debt/*.md)
    [ -f "$FILE" ] && exit 0  # allow edits to existing files
    echo "BLOCKED: Use 'reins doc create <type> \"title\"' to create harness docs."
    echo "Types: design, plan, spec, debt, retro, principle"
    exit 2
    ;;
  */harness/*/ideas/*.md)
    [ -f "$FILE" ] && exit 0
    echo "BLOCKED: Use 'reins idea \"text\"' or 'reins doc create idea \"title\"'."
    exit 2
    ;;
esac
```

**Key:** Blocks *creating new* docs but allows *editing existing* ones. Agents
scaffold with `reins doc create`, then fill in sections via Edit/Write.

### Skills Fork

**Source:** All 13 superpowers skill directories (~30 files)
**Destination:** `.claude/skills/` in the reins repo
**License:** MIT (Jesse Vincent) — include attribution

**Customization applied to all forked skills:**
- `docs/plans/` → `harness/{repo}/` (specific subdir per doc type)
- "save to file" instructions → "use `reins doc create <type>`"
- Add golden principles references where relevant
- Reference harness paths and reins CLI commands

**Deduplication with existing reins skills:**

| Superpowers | Reins | Resolution |
|---|---|---|
| `writing-plans` | `create-exec-plan` | Merge → uses `reins doc create plan` |
| `brainstorming` | — | Keep, uses `reins doc create design` |
| `requesting-code-review` | `review-pr` | Merge → unified review skill |
| — | `capture-idea` | Superseded by `reins doc create idea` |
| — | `publish-plan`, `sync-context`, `update-harness`, `analyze-codebase` | Keep as-is |

**New skill:** `using-reins` replaces `using-superpowers` — intro skill
establishing reins workflows and `reins doc create` as the doc creation
method.

### Migration

Move all existing harness docs into `harness/reins/`:

- `harness/design-docs/` → `harness/reins/design-docs/` (done)
- `harness/exec-plans/` → `harness/reins/exec-plans/` (done)
- `harness/ideas/` → `harness/reins/ideas/` (done)
- `harness/product-specs/` → `harness/reins/product-specs/` (done)
- `harness/tech-debt/` → `harness/reins/tech-debt/` (done)
- `harness/golden-principles.md` → `harness/reins/golden-principles.md` (done)
- `harness/quality-scores.md` → `harness/reins/quality-scores.md` (done)
- `harness/architecture/` → `harness/reins/architecture/` (done)

Update all cross-references, AGENTS.md path table, linter rules, and existing
skills.

### Hardcoded Path Fix

Replace all hardcoded `harness/` references in linter rules, bash hooks, and CI
workflows with `get_harness_dir()` helper. This was already identified as tech
debt (idea from 2026-02-23).

## Non-Goals

- Multi-harness support (one repo, multiple harnesses)
- Harness templates / flavors
- Upstream tracking for superpowers fork (deferred — add when we need to pull
  updates)
- `/init-project` agentic walkthrough skill (deferred to M2)
- Skill manifest / discoverability system (deferred)
- Document relevance tiers / search commands (deferred — good idea, separate
  design)

## Testing

- Unit tests for `reins doc create` (each type generates correct template,
  path, frontmatter)
- Unit test for repo slug derivation from various git remote URL formats
- Integration test: `reins doc create design "test"` → file exists at
  correct path with correct frontmatter
- Hook test: verify Write to new harness doc path is blocked, edit to existing
  is allowed
- Skills: manual verification that forked skills reference reins CLI

## Open Questions

- Should `reins doc create` auto-update the relevant index.md (e.g., add
  a row to `design-docs/index.md`)? Leaning yes.
- Should the enforcement hook also run on Bash tool calls that pipe content
  into harness doc paths? Edge case but possible.
