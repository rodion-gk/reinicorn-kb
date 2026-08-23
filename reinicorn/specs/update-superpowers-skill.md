---
type: spec
title: update-superpowers-skill
slug: update-superpowers-skill
lifecycle: done
status: implemented
created: 2026-03-26
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# update-superpowers-skill

## Problem

Reins forks 13 skills from the superpowers Claude Code plugin. When superpowers releases a new version, updating the forks is a manual process: copy files, re-add the `upstream:` frontmatter tag, re-apply local customizations (e.g. replacing `docs/superpowers/` paths with reins harness paths). This is error-prone and tedious.

## Design Goals

- One-command update flow: invoke the skill and it handles detection, copying, patching, and review
- Mechanical substitutions are defined in a config file, not left to agent memory
- Agent reviews results for anything the config didn't catch
- User approves before commit — no silent changes to skill content

## Design

### Skill Structure

```
.claude/skills/update-superpowers/
├── SKILL.md              # Agent instructions
└── replacements.yaml     # Mechanical substitutions
```

This skill lives in the reins repo only. It is a developer maintenance tool, not something that ships to projects using reins.

### replacements.yaml

Simple find/replace pairs applied to all forked SKILL.md files after copying from upstream. Literal string matching, longest match first to avoid partial replacements. Each entry has `find`, `replace`, and an optional `context` field for human readability.

### Agent Flow

**Step 1 — Detect versions:**
- Scan `~/.claude/plugins/cache/claude-plugins-official/superpowers/` for the latest version directory
- Read `upstream:` frontmatter from each forked skill to determine current version
- Report version delta. Exit early if already up to date.

**Step 2 — Identify what changed:**
- Diff each upstream skill SKILL.md against the corresponding fork (stripping the `upstream:` tag before comparison)
- Report summary: which skills changed, approximate size of changes
- Flag new upstream skills we don't have, or skills removed upstream

**Step 3 — Copy and patch:**
- For each changed skill: overwrite `.claude/skills/<name>/SKILL.md` with upstream content
- Apply all replacements from `replacements.yaml` (longest `find` string first)
- Inject `upstream: superpowers/<new-version>` into YAML frontmatter after the `description:` field
- Skip non-forked skills (`using-reins`, `populate-agents-md`, `update-superpowers`) entirely

**Step 4 — Agent review:**
- Read each updated file and check for:
  - Remaining `docs/superpowers/` references the replacements missed
  - Paths or conventions that don't match reins's harness structure
  - Structural changes that might conflict with how we use the skills
- Report findings to the user

**Step 5 — Pause for approval:**
- Show `git diff --stat` plus any findings from the review
- Wait for user to approve, request changes, or abort

**Step 6 — Update metadata and commit:**
- Update `ATTRIBUTION.md` with new version and date
- Commit all changes

## Non-Goals

- **Notification of new versions:** Detecting that superpowers has updated (e.g. at conversation start) is a separate concern. This skill handles the update itself, not the trigger.
- **Three-way merge:** If we make significant local content changes to forked skills in the future, we may need a more sophisticated merge strategy. For now, the forks are upstream-identical except for the mechanical replacements.
- **Shipping to users:** This skill is for reins developers only.
