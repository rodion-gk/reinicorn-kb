---
type: spec
title: Harness Auto-Load and Skill Triggers
slug: harness-auto-load-and-skill-triggers
lifecycle: active
status: draft
created: 2026-04-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Harness Auto-Load and Skill Triggers

## Problem

Two issues prevent reins from working reliably across platforms:

1. **No auto-load of AGENTS.md.** Only Claude Code natively reads AGENTS.md. Cursor, Copilot, and Gemini CLI each have their own instruction file (`.cursor/rules/`, `.github/copilot-instructions.md`, `GEMINI.md`). Without platform-specific files that reference AGENTS.md, those harnesses never see reins's project instructions.

2. **Skill trigger descriptions are too vague or missing keywords.** Several reins-specific skills (e.g., `using-reins`, `sync-context`, `publish-plan`) lack the keywords users actually say, causing the AI to skip skill invocation even when the user explicitly mentions "reins".

## Design Goals

- AGENTS.md remains the single source of truth for project instructions across all platforms.
- `reins init` generates platform-specific instruction files that point to AGENTS.md and include platform-adapted skill invocation guidance.
- Users choose which platforms to install for during init.
- Skill descriptions include specific, qualified trigger keywords that avoid false positives.

## Design

### Part A: Platform instruction files

**During `reins init`**, after copying skills, a new step prompts:

```
Which AI coding platforms do you use? (space to toggle, enter to confirm)
  [x] Claude Code
  [ ] Cursor
  [ ] GitHub Copilot
  [ ] Gemini CLI
```

Claude Code is pre-selected. For each selected platform, a platform-specific instruction file is generated from a bundled template.

| Platform | Generated file | Content |
|----------|---------------|---------|
| Claude Code | `CLAUDE.md` | Read AGENTS.md directive + skill invocation via `Skill` tool |
| Cursor | `.cursor/rules/reins.mdc` | Read AGENTS.md directive + skill invocation via slash commands |
| Copilot | `.github/copilot-instructions.md` | Read AGENTS.md directive + skill invocation via `skill` tool |
| Gemini CLI | `GEMINI.md` | Read AGENTS.md directive + skill invocation via `activate_skill` tool |

**Templates** live in `src/reins/_data/platform-instructions/{platform}.md` and are bundled via `pyproject.toml` `force-include`. Each uses `{repo}` substitution.

**Each file contains (~20-40 lines):**
1. Directive to read and follow AGENTS.md
2. Platform-adapted `using-reins` content (how skills work on that platform)
3. Doc creation rule (use `reins doc create`)

**Idempotency:** Skip if the file already exists, same as AGENTS.md.

**Init summary** updated to include generated files in the `git add` command.

### Part B: Skill trigger descriptions

Update 6 skill descriptions to include qualified trigger keywords:

| Skill | New description |
|-------|----------------|
| `using-reins` | "Use when starting any conversation in a reins-enabled repo, or when user mentions reins, harness docs, or project knowledge base. Establishes skill discovery and invocation patterns." |
| `populate-agents-md` | "Analyze repo and populate AGENTS.md through collaborative dialogue. Use when AGENTS.md contains the UNPOPULATED marker or when user asks to populate AGENTS.md." |
| `analyze-codebase` | "Use when attaching reins to an existing codebase or when user asks to analyze a codebase for reins setup. Runs automated analysis, then drafts harness docs for human review." |
| `sync-context` | "Use when user wants to pull latest harness state, says 'reins sync', or asks what changed in the harness. Reports changes and overlapping work from other branches." |
| `publish-plan` | "Use when user wants to push plan updates to the shared harness, says 'reins publish', or asks to share their plan. Rebases on main, pushes, reports conflicts." |
| `update-harness` | "Use when user asks to update quality scores, check for stale harness docs, refresh the tech-debt catalog, or run harness maintenance." |

Key principle: all trigger keywords are qualified with "reins" or "harness" context to avoid false positives on generic words like "sync" or "publish".

## Non-Goals

- Auto-syncing platform files when AGENTS.md changes (generate on init only; user re-runs if needed).
- Supporting platforms beyond the four listed above.
- Changing the AGENTS.md format or content.
