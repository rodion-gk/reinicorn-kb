---
type: idea
title: disable agent-local memory in favor of repo-tracked docs
slug: 2026-03-21-disable-agent-local-memory-in-favor-of-repo-tracked-docs
lifecycle: active
status: new
created: 2026-03-21
author: Michael Biehl
---

# disable agent-local memory in favor of repo-tracked docs

## Description

Agents like Claude Code have local memory systems (e.g., `~/.claude/projects/*/memory/`) that store project context outside the repo. This is worse than repo-tracked docs because:

- Not version-controlled or reviewable
- Not shared across machines or team members
- Duplicates what reins's harness already provides (ideas, designs, specs, etc.)
- Can go stale without anyone noticing

Reins should encourage (or enforce via hooks/skills) disabling agent-local memory in favor of using `reins <type> create` for all persistent project knowledge.

## Proposed Changes

1. **Skill guidance**: Skills should instruct agents to use `reins <type> create` instead of local memory when capturing ideas, project context, or feedback
2. **Init-time check**: `reins init` could detect and warn about existing agent memory directories, suggesting migration
3. **Documentation**: Getting-started guide should call this out as a best practice
